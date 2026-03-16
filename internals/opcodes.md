# Opcodes Reference

Complete list of io_uring operations as of kernel 6.19+. Extracted from `include/uapi/linux/io_uring.h` (liburing master, March 2026).

## I/O Operations

| Opcode | Value | Description | Since |
|--------|-------|-------------|-------|
| `IORING_OP_NOP` | 0 | No-op (benchmarking, testing) | 5.1 |
| `IORING_OP_READV` | 1 | Vectored read | 5.1 |
| `IORING_OP_WRITEV` | 2 | Vectored write | 5.1 |
| `IORING_OP_FSYNC` | 3 | File sync | 5.1 |
| `IORING_OP_READ_FIXED` | 4 | Read into registered buffer | 5.1 |
| `IORING_OP_WRITE_FIXED` | 5 | Write from registered buffer | 5.1 |
| `IORING_OP_READ` | 22 | Simple read (addr + len) | 5.6 |
| `IORING_OP_WRITE` | 23 | Simple write (addr + len) | 5.6 |
| `IORING_OP_READ_MULTISHOT` | 47 | Multishot read | 6.6 |
| `IORING_OP_READV_FIXED` | 58 | Vectored read into registered buffers | 6.15 |
| `IORING_OP_WRITEV_FIXED` | 59 | Vectored write from registered buffers | 6.15 |

## Networking

| Opcode | Value | Description | Since |
|--------|-------|-------------|-------|
| `IORING_OP_SENDMSG` | 9 | sendmsg(2) | 5.3 |
| `IORING_OP_RECVMSG` | 10 | recvmsg(2) | 5.3 |
| `IORING_OP_SEND` | 26 | send(2) | 5.6 |
| `IORING_OP_RECV` | 27 | recv(2) | 5.6 |
| `IORING_OP_ACCEPT` | 13 | accept4(2) | 5.5 |
| `IORING_OP_CONNECT` | 16 | connect(2) | 5.5 |
| `IORING_OP_SHUTDOWN` | 34 | shutdown(2) | 5.11 |
| `IORING_OP_SOCKET` | 45 | socket(2) | 5.19 |
| `IORING_OP_SEND_ZC` | 46 | Zero-copy send | 6.0 |
| `IORING_OP_SENDMSG_ZC` | 47 | Zero-copy sendmsg | 6.0 |
| `IORING_OP_BIND` | 54 | bind(2) | 6.14 |
| `IORING_OP_LISTEN` | 55 | listen(2) | 6.14 |
| `IORING_OP_RECV_ZC` | 56 | Zero-copy recv | 6.15 |

## Poll

| Opcode | Value | Description | Since |
|--------|-------|-------------|-------|
| `IORING_OP_POLL_ADD` | 6 | Add poll request | 5.1 |
| `IORING_OP_POLL_REMOVE` | 7 | Remove poll | 5.1 |
| `IORING_OP_EPOLL_CTL` | 29 | epoll_ctl via io_uring | 5.6 |
| `IORING_OP_EPOLL_WAIT` | 57 | epoll_wait via io_uring | 6.15 |

## File Operations

| Opcode | Value | Description | Since |
|--------|-------|-------------|-------|
| `IORING_OP_OPENAT` | 18 | openat(2) | 5.6 |
| `IORING_OP_OPENAT2` | 28 | openat2(2) | 5.6 |
| `IORING_OP_CLOSE` | 19 | close(2) | 5.6 |
| `IORING_OP_STATX` | 21 | statx(2) | 5.6 |
| `IORING_OP_FALLOCATE` | 17 | fallocate(2) | 5.6 |
| `IORING_OP_FTRUNCATE` | 54 | ftruncate(2) | 6.9 |
| `IORING_OP_RENAMEAT` | 35 | renameat2(2) | 5.11 |
| `IORING_OP_UNLINKAT` | 36 | unlinkat(2) | 5.11 |
| `IORING_OP_MKDIRAT` | 37 | mkdirat(2) | 5.15 |
| `IORING_OP_SYMLINKAT` | 38 | symlinkat(2) | 5.15 |
| `IORING_OP_LINKAT` | 39 | linkat(2) | 5.15 |
| `IORING_OP_FADVISE` | 24 | fadvise64(2) | 5.6 |
| `IORING_OP_MADVISE` | 25 | madvise(2) | 5.6 |
| `IORING_OP_SYNC_FILE_RANGE` | 8 | sync_file_range(2) | 5.2 |

## Extended Attributes

| Opcode | Value | Description | Since |
|--------|-------|-------------|-------|
| `IORING_OP_FSETXATTR` | 40 | fsetxattr(2) | 5.19 |
| `IORING_OP_SETXATTR` | 41 | setxattr(2) | 5.19 |
| `IORING_OP_FGETXATTR` | 42 | fgetxattr(2) | 5.19 |
| `IORING_OP_GETXATTR` | 43 | getxattr(2) | 5.19 |

## Splice/Pipe

| Opcode | Value | Description | Since |
|--------|-------|-------------|-------|
| `IORING_OP_SPLICE` | 30 | splice(2) | 5.7 |
| `IORING_OP_TEE` | 33 | tee(2) | 5.8 |
| `IORING_OP_PIPE` | 60 | pipe(2) | 6.14 |

## Buffer Management

| Opcode | Value | Description | Since |
|--------|-------|-------------|-------|
| `IORING_OP_PROVIDE_BUFFERS` | 31 | Provide buffers to pool | 5.7 |
| `IORING_OP_REMOVE_BUFFERS` | 32 | Remove buffers from pool | 5.7 |

## Timeouts

| Opcode | Value | Description | Since |
|--------|-------|-------------|-------|
| `IORING_OP_TIMEOUT` | 11 | Add timeout | 5.4 |
| `IORING_OP_TIMEOUT_REMOVE` | 12 | Cancel timeout | 5.4 |
| `IORING_OP_LINK_TIMEOUT` | 15 | Timeout for linked SQE | 5.5 |

## Control

| Opcode | Value | Description | Since |
|--------|-------|-------------|-------|
| `IORING_OP_ASYNC_CANCEL` | 14 | Cancel in-flight request | 5.5 |
| `IORING_OP_FILES_UPDATE` | 20 | Update registered files | 5.6 |
| `IORING_OP_MSG_RING` | 40 | Send message to another ring | 5.18 |
| `IORING_OP_FIXED_FD_INSTALL` | 53 | Install fixed fd into process | 6.7 |

## Futex

| Opcode | Value | Description | Since |
|--------|-------|-------------|-------|
| `IORING_OP_FUTEX_WAIT` | 51 | Async futex wait | 6.7 |
| `IORING_OP_FUTEX_WAKE` | 52 | Async futex wake | 6.7 |
| `IORING_OP_FUTEX_WAITV` | 53 | Vectored futex wait | 6.7 |

## Process

| Opcode | Value | Description | Since |
|--------|-------|-------------|-------|
| `IORING_OP_WAITID` | 50 | waitid(2) | 6.7 |

## Passthrough

| Opcode | Value | Description | Since |
|--------|-------|-------------|-------|
| `IORING_OP_URING_CMD` | 46 | Device-specific command | 5.19 |
| `IORING_OP_NOP128` | 61 | 128-byte NOP (for mixed SQE rings) | 6.18 |
| `IORING_OP_URING_CMD128` | 62 | 128-byte command variant | 6.18 |

## SQE Flags

Applied via `sqe->flags`:

| Flag | Description |
|------|-------------|
| `IOSQE_FIXED_FILE` | fd is a registered file index |
| `IOSQE_IO_DRAIN` | Serialize with previous SQEs |
| `IOSQE_IO_LINK` | Link to next SQE (cancel chain on failure) |
| `IOSQE_IO_HARDLINK` | Link to next SQE (continue chain on failure) |
| `IOSQE_ASYNC` | Force async execution (skip fast path) |
| `IOSQE_BUFFER_SELECT` | Use provided buffer from buf_group |
| `IOSQE_CQE_SKIP_SUCCESS` | Don't generate CQE on success |

## Multishot Variants

Several ops support multishot mode (one SQE → multiple CQEs):

- **Accept**: `IORING_ACCEPT_MULTISHOT` via ioprio
- **Recv**: `IORING_RECV_MULTISHOT` via ioprio  
- **Poll**: `IORING_POLL_ADD_MULTI` via len
- **Timeout**: `IORING_TIMEOUT_MULTISHOT` via timeout_flags
- **Read**: `IORING_OP_READ_MULTISHOT` (dedicated opcode)

CQEs from multishot ops have `IORING_CQE_F_MORE` set until the final one.
