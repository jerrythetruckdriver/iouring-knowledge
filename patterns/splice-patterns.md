# Splice Patterns

Zero-copy data movement between file descriptors using io_uring.

## Opcodes

| Op | Opcode | Since | Description |
|----|--------|-------|-------------|
| `IORING_OP_SPLICE` | 30 | 5.7 | Move data between two fds via pipe |
| `IORING_OP_TEE` | 33 | 5.8 | Duplicate pipe data without consuming |

## SQE Layout (Splice)

```
sqe->fd          = fd_out
sqe->splice_fd_in = fd_in          // or fixed fd with SPLICE_F_FD_IN_FIXED
sqe->off         = off_out         // offset in fd_out (-1 for pipe)
sqe->splice_off_in = off_in        // offset in fd_in (-1 for pipe)
sqe->len         = nbytes
sqe->splice_flags = SPLICE_F_MOVE | SPLICE_F_MORE | ...
```

For fixed file descriptors on the input side:
```c
sqe->splice_flags |= SPLICE_F_FD_IN_FIXED;
```

## SQE Layout (Tee)

```
sqe->fd          = fd_out          // must be pipe
sqe->splice_fd_in = fd_in          // must be pipe
sqe->len         = nbytes
sqe->splice_flags = flags
```

## Core Constraint

Splice requires at least one end to be a pipe. You can't splice socket→file directly. The pipe is the kernel buffer that enables zero-copy.

```
file → pipe → socket    (sendfile equivalent)
socket → pipe → file    (receive-to-disk)
pipe → pipe             (tee: duplicate)
```

## Pattern: File → Socket (Async sendfile)

```c
// Create pipe
int pipefd[2];
pipe(pipefd);  // or use IORING_OP_PIPE (6.14) for async creation

// SQE 1: splice file → pipe
io_uring_prep_splice(sqe1, file_fd, file_offset, pipefd[1], -1, chunk_size, 0);
sqe1->flags |= IOSQE_IO_LINK;

// SQE 2: splice pipe → socket
io_uring_prep_splice(sqe2, pipefd[0], -1, socket_fd, -1, chunk_size, 0);
```

Linked submission ensures ordering. The kernel moves pages by reference — no userspace copies.

## Pattern: Socket → File (Receive to Disk)

```c
// SQE 1: splice socket → pipe
io_uring_prep_splice(sqe1, socket_fd, -1, pipefd[1], -1, chunk_size, 0);
sqe1->flags |= IOSQE_IO_LINK;

// SQE 2: splice pipe → file
io_uring_prep_splice(sqe2, pipefd[0], -1, file_fd, file_offset, chunk_size, 0);
```

## Pattern: Proxy (Socket → Socket)

```c
// SQE 1: splice client → pipe
io_uring_prep_splice(sqe1, client_fd, -1, pipefd[1], -1, BUF_SIZE, 0);
sqe1->flags |= IOSQE_IO_LINK;

// SQE 2: splice pipe → upstream
io_uring_prep_splice(sqe2, pipefd[0], -1, upstream_fd, -1, BUF_SIZE, 0);
```

Proxy pattern avoids touching data in userspace entirely. Good for reverse proxies, load balancers, tunnels.

## Pattern: Tee + Splice (Log and Forward)

```c
// Tee: duplicate pipe data
io_uring_prep_tee(sqe1, pipefd[0], log_pipe[1], BUF_SIZE, 0);
sqe1->flags |= IOSQE_IO_LINK;

// Splice: forward original
io_uring_prep_splice(sqe2, pipefd[0], -1, upstream_fd, -1, BUF_SIZE, 0);
sqe2->flags |= IOSQE_IO_LINK;

// Splice: log copy to file
io_uring_prep_splice(sqe3, log_pipe[0], -1, log_fd, -1, BUF_SIZE, 0);
```

## Pattern: Fully Async with IORING_OP_PIPE (6.14+)

```c
// SQE 1: create pipe asynchronously
io_uring_prep_pipe(sqe1, 0);
sqe1->file_index = IORING_FILE_INDEX_ALLOC;  // auto-allocate fixed fds
sqe1->flags |= IOSQE_IO_LINK;

// SQE 2-3: splice through the pipe (using fixed fd from pipe result)
// ... link chain continues
```

## Splice vs Zero-Copy Send/Recv

| | Splice | ZC Send (6.0) | zcrx (6.15) |
|---|--------|---------------|-------------|
| Requires pipe | Yes | No | No |
| Network only | No (any fd) | Yes (socket) | Yes (NIC) |
| Kernel version | 5.7 | 6.0 | 6.15 |
| Best for | File↔socket, proxying | High-throughput sends | NIC→userspace |
| Overhead | Pipe management | Notification CQE | Hardware setup |

Splice is the general-purpose zero-copy mechanism. ZC send/recv are specialized for networking. Use splice when you need file-to-socket or socket-to-socket movement. Use ZC send when you already have the data in a buffer and want to avoid the copy to kernel.

## Gotchas

1. **Pipe buffer limit**: Default 64KB (16 pages). Tune with `fcntl(F_SETPIPE_SZ)` for larger transfers.
2. **Short splices**: CQE res may be less than requested. Handle partial transfers.
3. **Pipe lifecycle**: Each proxy connection needs its own pipe pair. Pool them.
4. **Fixed fd optimization**: Use `SPLICE_F_FD_IN_FIXED` to avoid fd lookup overhead on hot paths.
5. **EAGAIN**: Non-blocking sockets may return -EAGAIN on the socket end. Arm poll or retry.
