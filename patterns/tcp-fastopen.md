# TCP Fast Open (TFO) and io_uring

## How It Works

TCP Fast Open sends data in the SYN packet, eliminating one RTT for repeat connections. The kernel handles TFO transparently — io_uring doesn't need special support.

## Server Side

```c
// Enable TFO on listening socket
int qlen = 5; // TFO queue length
setsockopt(fd, SOL_TCP, TCP_FASTOPEN, &qlen, sizeof(qlen));

// Then normal io_uring accept — TFO is transparent
io_uring_prep_multishot_accept(sqe, fd, NULL, NULL, 0);

// Or set via URING_CMD (6.11+)
io_uring_prep_cmd_sock(sqe, SOCKET_URING_OP_SETSOCKOPT,
                       fd, SOL_TCP, TCP_FASTOPEN, &qlen, sizeof(qlen));
```

TFO data arrives with the SYN. The kernel buffers it. First `recv` after `accept` gets the TFO payload — no io_uring changes needed.

## Client Side

```c
// Method 1: sendmsg with MSG_FASTOPEN
struct msghdr msg = { .msg_name = &addr, .msg_namelen = addrlen, ... };
io_uring_prep_sendmsg(sqe, fd, &msg, MSG_FASTOPEN);
// Combines connect + first send into one SQE

// Method 2: TCP_FASTOPEN_CONNECT setsockopt
int enable = 1;
setsockopt(fd, SOL_TCP, TCP_FASTOPEN_CONNECT, &enable, sizeof(enable));
// Then normal connect + send — kernel handles TFO transparently
io_uring_prep_connect(sqe, fd, &addr, addrlen);
// ... link to first send
```

## io_uring-Specific Patterns

**Linked TFO client chain:**
```c
// SQE 1: sendmsg with MSG_FASTOPEN (connect + first data)
io_uring_prep_sendmsg(sqe1, fd, &msg, MSG_FASTOPEN);
sqe1->flags |= IOSQE_IO_LINK;

// SQE 2: multishot recv for response
io_uring_prep_recv_multishot(sqe2, fd, NULL, 0, 0);
sqe2->flags |= IOSQE_BUFFER_SELECT;
sqe2->buf_group = bgid;
```

One submission, connection established with data, then continuous receive. Two SQEs for what would be 3+ syscalls (connect, send, recv loop).

**Batch TFO connections:**
```c
// Submit N TFO connections in one batch
for (int i = 0; i < N; i++) {
    sqe = io_uring_get_sqe(ring);
    io_uring_prep_sendmsg(sqe, fds[i], &msgs[i], MSG_FASTOPEN);
}
io_uring_submit(ring);
// All N connections initiated with one syscall
```

## TFO Cookie Handling

TFO cookies are managed entirely by the kernel. First connection to a server does a normal 3-way handshake and caches the cookie. Subsequent connections use the cached cookie to send data in the SYN. io_uring doesn't see any of this.

## When TFO + io_uring Matters

- **HTTP client making many short-lived connections** — saves 1 RTT per reconnection
- **CDN edge → origin** — connection churn with repeated destinations
- **DNS over TCP** — short queries, frequent reconnections

## Limitations

- TFO replay attacks — same data can be replayed by middleboxes. Don't use for non-idempotent first requests.
- Many middleboxes strip TCP options, breaking TFO silently.
- Cookie cache is per-destination IP. Not per-port.

## See Also

- [Socket Lifecycle](socket-lifecycle.md) — full async socket chain
- [Batch Connect](batch-connect.md) — outbound connection patterns
- [TCP CORK/NODELAY](tcp-cork-nodelay.md) — TCP tuning with io_uring
