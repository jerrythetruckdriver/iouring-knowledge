# sendfile() Equivalence — Splice vs Zero-Copy Send

## The Problem

`sendfile()` copies file data to a socket with zero (or one) copy. io_uring has no `IORING_OP_SENDFILE`. Instead, you have two paths.

## Path 1: Splice via Pipe (io_uring equivalent of sendfile)

```
SPLICE: file_fd → pipe → socket_fd
```

Two linked SPLICE operations through a kernel pipe:

```c
// SQE 1: file → pipe (IOSQE_IO_LINK)
sqe->opcode = IORING_OP_SPLICE;
sqe->splice_fd_in = file_fd;     // source
sqe->fd = pipe_wr;               // dest (pipe write end)
sqe->len = chunk_size;
sqe->splice_off_in = file_offset;
sqe->flags = IOSQE_IO_LINK;

// SQE 2: pipe → socket
sqe->opcode = IORING_OP_SPLICE;
sqe->splice_fd_in = pipe_rd;     // source (pipe read end)
sqe->fd = socket_fd;             // dest
sqe->len = chunk_size;
sqe->splice_flags = SPLICE_F_NONBLOCK;
```

**Kernel versions:** SPLICE since 5.7, PIPE since 6.14 (async pipe creation).

**Pros:** Works with any socket type, works with regular files, true zero-copy from page cache.

**Cons:** Requires a pipe (extra fd pair), two SQEs per chunk, pipe buffer limits (~1MB default).

## Path 2: SEND_ZC with Registered Buffers (direct zero-copy)

```
READ file → registered buffer → SEND_ZC to socket
```

```c
// SQE 1: read file into registered buffer (IOSQE_IO_LINK)
sqe->opcode = IORING_OP_READ_FIXED;
sqe->fd = file_fd;
sqe->buf_index = buf_idx;
sqe->len = chunk_size;
sqe->flags = IOSQE_IO_LINK;

// SQE 2: zero-copy send from registered buffer
sqe->opcode = IORING_OP_SEND_ZC;
sqe->fd = socket_fd;
sqe->buf_index = buf_idx;
sqe->msg_flags = 0;
sqe->ioprio = IORING_RECVSEND_FIXED_BUF;
```

**Kernel versions:** SEND_ZC since 6.0, notification CQE for buffer lifetime.

**Pros:** No pipe needed, works with registered buffers, can batch multiple sends.

**Cons:** Two CQEs per send (data + notification), buffer can't be reused until notification CQE arrives, requires NIC support for true zero-copy (otherwise falls back to copy).

## Path 3: SENDMSG_ZC (vectored zero-copy)

Same as SEND_ZC but with `struct msghdr` for scatter-gather. Useful when sending file data + protocol headers in one operation.

## Comparison

| | sendfile() | Splice | SEND_ZC |
|---|-----------|--------|---------|
| Syscalls | 1 | 1 (batched) | 1 (batched) |
| Extra fds | None | Pipe pair | None |
| Zero-copy | Page cache → NIC | Page cache → NIC | Buffer → NIC |
| File types | Regular files | Any fd | Registered buffers |
| Batching | No | Yes (linked) | Yes (linked) |
| CQEs | N/A | 2 | 2 (data + notif) |

## recvmmsg / sendmmsg Equivalence

**Bundle operations** (6.10) replace `recvmmsg()`/`sendmmsg()`:

```c
// One SQE, multiple buffers consumed/sent
sqe->opcode = IORING_OP_RECV;
sqe->ioprio = IORING_RECVSEND_BUNDLE;
sqe->buf_group = bg_id;          // provided buffer ring
sqe->flags |= IOSQE_BUFFER_SELECT;
```

One SQE → one CQE reporting how many buffers were filled. With `IORING_SEND_VECTORIZED`, send gathers from multiple provided buffers in order.

**vs sendmmsg():** sendmmsg batches N messages into 1 syscall. Bundle ops batch N buffer fills into 1 SQE+CQE pair. Bundle is strictly better — less SQ/CQ pressure, works with multishot.

## When to Use What

- **Static file serving:** Splice (page cache → socket, true zero-copy, no buffer management)
- **Dynamic content:** SEND_ZC with registered buffers (data already in your buffers)
- **High message rate:** Bundle ops (UDP, many small messages)
- **Mixed headers + file data:** SENDMSG_ZC with iovec
