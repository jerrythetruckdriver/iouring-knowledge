# Full Async Socket Lifecycle

Every socket operation — from creation to close — can be async through io_uring. No syscalls required after ring setup.

## The Full Chain (6.14+)

```
SOCKET → BIND → LISTEN → ACCEPT → RECV → SEND → SHUTDOWN → CLOSE
  44       54     55       13      27     26       34        19
```

All opcodes. All async. All zero-syscall on the data path.

## Opcode Reference

| Operation | Opcode | Since | Notes |
|-----------|--------|-------|-------|
| `SOCKET` | 44 | 5.19 | Create socket fd |
| `BIND` | 54 | 6.14 | Bind to address |
| `LISTEN` | 55 | 6.14 | Start listening |
| `ACCEPT` | 13 | 5.5 | Accept connection (multishot: 5.19) |
| `CONNECT` | 16 | 5.5 | Client connect |
| `RECV` | 27 | 5.6 | Receive data (multishot: 6.0) |
| `RECVMSG` | 10 | 5.3 | Receive with ancillary data |
| `SEND` | 26 | 5.6 | Send data |
| `SENDMSG` | 9 | 5.3 | Send with ancillary data |
| `SEND_ZC` | 46 | 6.0 | Zero-copy send |
| `SENDMSG_ZC` | 47 | 6.0 | Zero-copy sendmsg |
| `RECV_ZC` | 56 | 6.15 | Zero-copy receive (zcrx) |
| `SHUTDOWN` | 34 | 5.11 | Shutdown socket |
| `CLOSE` | 19 | 5.6 | Close fd |

## Pattern: Server — Zero Syscall After Setup

```c
// Phase 1: Setup (can also be async via linked chain)
io_uring_prep_socket(sqe, AF_INET, SOCK_STREAM, 0, 0);
sqe->file_index = IORING_FILE_INDEX_ALLOC;  // direct descriptor
sqe->flags |= IOSQE_IO_LINK;

io_uring_prep_bind(sqe, /*fd from prev*/, &addr, addrlen);
sqe->flags |= IOSQE_IO_LINK | IOSQE_FIXED_FILE;

io_uring_prep_listen(sqe, /*fd*/, backlog);
sqe->flags |= IOSQE_FIXED_FILE;

io_uring_submit(&ring);

// Phase 2: Accept loop (multishot — one SQE, unlimited accepts)
io_uring_prep_multishot_accept(sqe, listen_fd, NULL, NULL, 0);
sqe->flags |= IOSQE_FIXED_FILE;
sqe->file_index = IORING_FILE_INDEX_ALLOC;  // auto-allocate direct desc

// Phase 3: Per-connection recv (multishot + provided buffers)
io_uring_prep_recv_multishot(sqe, client_fd, NULL, 0, 0);
sqe->flags |= IOSQE_BUFFER_SELECT | IOSQE_FIXED_FILE;
sqe->buf_group = my_buf_group;
```

After initial setup, the entire server runs on CQE processing. No `accept()`, no `recv()`, no `send()` syscalls.

## Pattern: Client Connection

```c
// Linked chain: socket → connect → send → recv
io_uring_prep_socket(sqe1, AF_INET, SOCK_STREAM, 0, 0);
sqe1->file_index = IORING_FILE_INDEX_ALLOC;
sqe1->flags |= IOSQE_IO_LINK;

io_uring_prep_connect(sqe2, /*fd*/, &server_addr, addrlen);
sqe2->flags |= IOSQE_IO_LINK | IOSQE_FIXED_FILE;

io_uring_prep_send(sqe3, /*fd*/, request, req_len, 0);
sqe3->flags |= IOSQE_IO_LINK | IOSQE_FIXED_FILE;

io_uring_prep_recv(sqe4, /*fd*/, response, resp_len, 0);
sqe4->flags |= IOSQE_FIXED_FILE;

io_uring_submit(&ring);  // One submit, four operations
```

## Direct Descriptors

Use `IORING_FILE_INDEX_ALLOC` with socket/accept to get direct descriptors (registered in the fixed file table automatically). Benefits:

- No fd table lookup on each operation
- No `close()` syscall (use `IORING_OP_CLOSE` with `IOSQE_FIXED_FILE`)
- No fd exhaustion concerns (independent of process fd limit)

Install to normal fd space if needed:
```c
io_uring_prep_fixed_fd_install(sqe, direct_fd, 0);
```

## Socket Options via URING_CMD (6.11+)

```c
// Get bytes available
sqe->opcode = IORING_OP_URING_CMD;
sqe->cmd_op = SOCKET_URING_OP_SIOCINQ;

// Set socket option
sqe->opcode = IORING_OP_URING_CMD;
sqe->cmd_op = SOCKET_URING_OP_SETSOCKOPT;
sqe->level = SOL_SOCKET;
sqe->optname = SO_RCVLOWAT;
sqe->optval = &val;
sqe->optlen = sizeof(val);
```

SO_RCVLOWAT integration with multishot recv: set a low watermark so multishot recv only fires when there's enough data. Reduces CQE churn for small packets.

## Multishot Accept + Recv Combo

The high-performance server pattern:

```c
// One SQE: accept connections forever
io_uring_prep_multishot_accept(sqe, listen_fd, NULL, NULL, 0);
// CQE has CQE_F_MORE flag — more coming

// For each accepted connection:
//   CQE flags >> 16 = buffer ID (if using provided buffers)
//   CQE res = new fd (or fixed fd index)

// Arm multishot recv on accepted fd
io_uring_prep_recv_multishot(sqe, new_fd, NULL, 0, 0);
sqe->flags |= IOSQE_BUFFER_SELECT;
sqe->buf_group = bgid;
// CQE per incoming data chunk, CQE_F_MORE until disconnect
```

Two SQEs total for unlimited connections with unlimited receives. The kernel does the rest.

## Bundle Recv/Send (6.10+)

```c
io_uring_prep_recv(sqe, fd, NULL, 0, 0);
sqe->ioprio |= IORING_RECVSEND_BUNDLE;
sqe->flags |= IOSQE_BUFFER_SELECT;
sqe->buf_group = bgid;
// CQE res = number of buffers filled (contiguous from buf_id in flags)
```

Batch multiple buffers in one CQE. Reduces CQ pressure on high-throughput connections.

## Graceful Shutdown

```c
// Shutdown write side
io_uring_prep_shutdown(sqe, client_fd, SHUT_WR);
sqe->flags |= IOSQE_IO_LINK | IOSQE_FIXED_FILE;

// Close fd
io_uring_prep_close_direct(sqe, fixed_fd_idx);
```

## The Syscall Comparison

| Operation | Traditional | io_uring |
|-----------|------------|----------|
| Create socket | `socket()` — 1 syscall | Part of batch |
| Bind | `bind()` — 1 syscall | Part of batch |
| Listen | `listen()` — 1 syscall | Part of batch |
| Accept | `accept4()` per conn — N syscalls | 1 SQE (multishot) |
| Recv | `recv()` per read — N×M syscalls | 1 SQE per conn (multishot) |
| Send | `send()` per write — N×M syscalls | 1 SQE per send |
| Close | `close()` per fd — N syscalls | Part of batch |

A server handling 10K connections doing 100 ops/sec each: 1M+ syscalls/sec with traditional sockets. With multishot io_uring: a few hundred SQEs total after setup.
