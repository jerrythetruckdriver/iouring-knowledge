# Networking Patterns

## TCP Server Lifecycle (io_uring native)

With io_uring, the entire TCP lifecycle stays in the ring:

```
SOCKET → BIND → LISTEN → ACCEPT (multishot) → RECV (multishot) → SEND → CLOSE
```

All of these are io_uring opcodes since kernel 6.11. No syscalls needed after ring setup.

## Multishot Accept

One SQE, unlimited accepts. Set `IORING_ACCEPT_MULTISHOT` in `sqe->ioprio`:

```c
io_uring_prep_multishot_accept(sqe, listen_fd, NULL, NULL, 0);
```

Each accepted connection produces a CQE with `IORING_CQE_F_MORE` set. New fd in `cqe->res`. Final CQE (error or cancellation) has `IORING_CQE_F_MORE` cleared.

With `IORING_FILE_INDEX_ALLOC` + `IORING_REGISTER_FILE_ALLOC_RANGE`, accepted fds go directly into the fixed file table. Zero fd allocation overhead.

## Multishot Recv

One SQE, multiple receives. Requires provided buffer ring:

```c
io_uring_prep_recv_multishot(sqe, fd, NULL, 0, 0);
sqe->buf_group = buffer_group_id;
sqe->flags |= IOSQE_BUFFER_SELECT;
```

Each recv produces a CQE. Buffer ID in upper 16 bits of `cqe->flags` (when `IORING_CQE_F_BUFFER` set). Return buffer to ring after processing.

`IORING_CQE_F_SOCK_NONEMPTY` hints more data available — useful for batching.

## Zero-Copy Send

Avoids copying send data to kernel:

```c
io_uring_prep_send_zc(sqe, fd, buf, len, 0, 0);
```

Two CQEs per operation:
1. Completion CQE — send result
2. Notification CQE (`IORING_CQE_F_NOTIF`) — safe to reuse/free buffer

Don't touch the buffer until you see the notification CQE.

`IORING_SEND_ZC_REPORT_USAGE` reports if data was actually zero-copied or fell back to copy (`IORING_NOTIF_USAGE_ZC_COPIED`).

## Zero-Copy Receive (6.12+)

True zero-copy RX path using NIC hardware:

```c
// Register interface queue
struct io_uring_zcrx_ifq_reg reg = {
    .if_idx = netdev_index,
    .if_rxq = rx_queue,
    .rq_entries = 4096,
};
io_uring_register(fd, IORING_REGISTER_ZCRX_IFQ, &reg, 1);
```

Data arrives in NIC-managed memory, mapped into userspace. Refill queue returns buffers to NIC. DMA-buf support via `IORING_ZCRX_AREA_DMABUF`.

## Bundle Send/Recv (6.10+)

Send or receive multiple buffers in one operation:

```c
sqe->ioprio |= IORING_RECVSEND_BUNDLE;
sqe->flags |= IOSQE_BUFFER_SELECT;
sqe->buf_group = group_id;
```

Grabs multiple buffers from the provided buffer group. `cqe->res` = number of buffers consumed. Contiguous buffer IDs starting from the ID in `cqe->flags`.

## NAPI Busy Polling (6.9+)

Direct kernel networking stack integration:

```c
struct io_uring_napi napi = {
    .busy_poll_to = 100,  // microseconds
    .prefer_busy_poll = 1,
};
io_uring_register(fd, IORING_REGISTER_NAPI, &napi, 1);
```

Polls the NIC directly for packets. Lowest possible latency for network I/O. Pairs well with `IOPOLL` mode.

## Socket Options via URING_CMD

Get/set socket options without syscalls:

```c
sqe->opcode = IORING_OP_URING_CMD;
sqe->cmd_op = SOCKET_URING_OP_SETSOCKOPT;
sqe->level = SOL_SOCKET;
sqe->optname = SO_REUSEADDR;
```

Available ops: `SIOCINQ`, `SIOCOUTQ`, `GETSOCKOPT`, `SETSOCKOPT`, `TX_TIMESTAMP`, `GETSOCKNAME`.
