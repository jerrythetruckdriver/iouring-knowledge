# MSG_WAITALL, MSG_DONTWAIT, and Other Flags

## Flag Passthrough

io_uring send/recv operations pass `msg_flags` through to the kernel socket layer. Most standard flags work as expected, with some io_uring-specific twists.

## MSG_WAITALL

```c
io_uring_prep_recv(sqe, fd, buf, len, MSG_WAITALL);
```

Blocks (async-waits) until the full `len` bytes are received or the connection closes/errors. Without it, recv returns whatever's available.

### Interaction with io_uring

- **Normal recv**: MSG_WAITALL works as expected. CQE arrives when `len` bytes are ready or error/EOF.
- **Multishot recv**: MSG_WAITALL is **ignored**. Multishot delivers whatever's available per event — that's the point.
- **Provided buffers**: MSG_WAITALL with provided buffers works but the buffer must be ≥ `len`. If the selected buffer is smaller, you get a short read.
- **Bundle recv**: MSG_WAITALL doesn't apply — bundles consume multiple buffers.

### When to Use

Protocol framing with fixed-size headers:

```c
// Read exactly 4-byte length prefix
io_uring_prep_recv(sqe, fd, &header, 4, MSG_WAITALL);
```

For variable-length bodies, use multishot recv with provided buffers instead.

## MSG_DONTWAIT

```c
io_uring_prep_send(sqe, fd, buf, len, MSG_DONTWAIT);
```

Non-blocking: return -EAGAIN immediately if the operation would block.

### Mostly Irrelevant for io_uring

io_uring already handles async I/O. A normal send/recv SQE will poll-arm and complete when ready. MSG_DONTWAIT is only useful if you want to distinguish "no data available right now" from "waiting for data" — and with io_uring, you usually don't care about that distinction.

Exception: `IORING_ACCEPT_DONTWAIT` (in ioprio, not msg_flags) is useful for one-shot accept checks.

## MSG_MORE

```c
io_uring_prep_send(sqe, fd, data, len, MSG_MORE);
```

Tells the kernel more data follows — don't push a partial TCP segment yet. Equivalent to TCP_CORK for a single send. See [tcp-cork-nodelay.md](tcp-cork-nodelay.md) for the full pattern.

## MSG_NOSIGNAL

```c
io_uring_prep_send(sqe, fd, buf, len, MSG_NOSIGNAL);
```

Suppress SIGPIPE on broken connection. **Always set this.** io_uring delivers errors via CQE (res = -EPIPE), not signals. There's no reason to ever receive SIGPIPE from an io_uring send.

## MSG_ZEROCOPY

Don't set `MSG_ZEROCOPY` in msg_flags. Use `IORING_OP_SEND_ZC` / `IORING_OP_SENDMSG_ZC` instead — these set up proper zero-copy notification handling.

## MSG_TRUNC / MSG_PEEK

```c
// Peek at data without consuming
io_uring_prep_recv(sqe, fd, buf, len, MSG_PEEK);

// Get truncated message length for datagrams
io_uring_prep_recv(sqe, fd, buf, len, MSG_TRUNC);
```

Both work with io_uring. MSG_PEEK is useful for protocol detection (peek at first bytes, then decide which handler to use). MSG_TRUNC with provided buffers lets you detect oversized datagrams.

## MSG_EOR (End of Record)

For SCTP and UNIX domain sockets (SOCK_SEQPACKET). Works in io_uring — CQE reflects the flag in the result.

## Flag Compatibility Table

| Flag | send | recv | multishot recv | send_zc | Notes |
|------|------|------|----------------|---------|-------|
| MSG_WAITALL | n/a | ✅ | Ignored | n/a | Full-length recv |
| MSG_DONTWAIT | ✅ | ✅ | ✅ | ✅ | Mostly redundant |
| MSG_MORE | ✅ | n/a | n/a | ✅ | Cork equivalent |
| MSG_NOSIGNAL | ✅ | n/a | n/a | ✅ | Always set for send |
| MSG_PEEK | n/a | ✅ | ❌ | n/a | Read without consume |
| MSG_TRUNC | n/a | ✅ | ✅ | n/a | Datagram truncation |
| MSG_ZEROCOPY | ❌ | n/a | n/a | n/a | Use SEND_ZC ops |
| MSG_EOR | ✅ | ✅ | ✅ | ✅ | SCTP/seqpacket |

## Recommendation

For most io_uring networking:

```c
// Send: always MSG_NOSIGNAL, add MSG_MORE for batching
io_uring_prep_send(sqe, fd, buf, len, MSG_NOSIGNAL);

// Recv: multishot with provided buffers, no flags needed
io_uring_prep_recv_multishot(sqe, fd, NULL, 0, 0);
sqe->flags |= IOSQE_BUFFER_SELECT;
sqe->buf_group = my_group;
```

Let io_uring handle the async. Don't fight it with synchronous-era flags.
