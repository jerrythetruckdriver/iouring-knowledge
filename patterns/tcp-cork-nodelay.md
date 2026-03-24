# TCP_CORK, TCP_NODELAY, and io_uring

## The Problem

TCP_CORK and TCP_NODELAY control Nagle's algorithm and cork behavior — when the kernel flushes partial TCP segments. With synchronous I/O, you'd toggle these around writes. With io_uring, the story changes.

## Socket Options via URING_CMD

Since 6.11, socket options are async:

```c
// SETSOCKOPT via URING_CMD
io_uring_prep_cmd_sock(sqe, SOCKET_URING_OP_SETSOCKOPT, fd,
                       SOL_TCP, TCP_NODELAY, &val, sizeof(val));

// GETSOCKOPT
io_uring_prep_cmd_sock(sqe, SOCKET_URING_OP_GETSOCKOPT, fd,
                       SOL_TCP, TCP_CORK, &val, sizeof(val));
```

## Patterns

### TCP_NODELAY (Most Common)

Set once at connection creation. Link it after accept:

```
ACCEPT → [link] → SETSOCKOPT(TCP_NODELAY=1)
```

With direct descriptors, this is zero syscalls total.

### TCP_CORK (Batch Writes)

Classic pattern: cork → write headers → write body → uncork. With io_uring:

```
SETSOCKOPT(TCP_CORK=1) → [link] → SEND(headers) → [link] → SEND(body) → [link] → SETSOCKOPT(TCP_CORK=0)
```

The entire sequence submits as one linked chain. One io_uring_enter() call.

### Better Alternative: MSG_MORE

Skip CORK entirely. Use `msg_flags = MSG_MORE` on intermediate sends:

```c
// SQE 1: headers (more data coming)
io_uring_prep_send(sqe1, fd, headers, hdr_len, MSG_MORE);

// SQE 2: body (final segment, no MSG_MORE)
io_uring_prep_send(sqe2, fd, body, body_len, 0);
```

No socket option toggling. No linked chain dependency. Independent SQEs that can batch naturally.

### Even Better: Vectored Send

```c
struct iovec iov[2] = {
    { .iov_base = headers, .iov_len = hdr_len },
    { .iov_base = body,    .iov_len = body_len },
};
io_uring_prep_sendmsg(sqe, fd, &msg, 0);
```

One SQE, one CQE, kernel coalesces into optimal segments.

### Best: Zero-Copy Vectored Send

```c
io_uring_prep_sendmsg_zc(sqe, fd, &msg, 0);
sqe->ioprio |= IORING_SEND_VECTORIZED;
```

Zero-copy + vectored + one SQE. The endgame.

## What Doesn't Work

- **TCP_CORK across submit batches**: Cork set in batch N, uncork in batch N+1. The kernel may flush between submits. Use MSG_MORE or vectored sends instead.
- **CORK with multishot recv**: Recv doesn't care about cork. Cork affects send path only.

## Recommendation

| Need | Approach |
|------|----------|
| Low-latency responses | TCP_NODELAY=1 at accept time, forget about it |
| Batch small writes | MSG_MORE flag on non-final sends |
| Headers + body | sendmsg (vectored) or sendmsg_zc |
| Complex multi-part | Linked CORK→sends→UNCORK chain |

MSG_MORE > TCP_CORK for io_uring workloads. Fewer SQEs, no option toggling, no link chain overhead.
