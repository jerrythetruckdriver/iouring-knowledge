# TCP Keepalive Patterns

## The Problem

TCP keepalive detects dead connections. Traditionally configured with setsockopt and monitored passively by the kernel. io_uring can handle the setsockopt setup and integrate keepalive-triggered events into your event loop.

## Setting Keepalive via io_uring

Socket options via URING_CMD (6.11+):

```c
// Enable keepalive on a connected socket
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);

// SETSOCKOPT via URING_CMD
// op = IO_URING_SOCKET_OP_SETSOCKOPT
// level = SOL_SOCKET, optname = SO_KEEPALIVE
// value = 1
io_uring_prep_cmd_sock(sqe, SOCKET_URING_OP_SETSOCKOPT,
                       fd, SOL_SOCKET, SO_KEEPALIVE, &val, sizeof(val));
```

Keepalive parameters (all can be set async via URING_CMD):
- `TCP_KEEPIDLE` — seconds before first probe (default 7200)
- `TCP_KEEPINTVL` — seconds between probes (default 75)
- `TCP_KEEPCNT` — number of probes before giving up (default 9)

## Monitoring Dead Connections

### Option 1: Multishot POLL_ADD

```c
// Watch for POLLERR/POLLHUP on each connection
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_poll_multishot(sqe, fd, POLLERR | POLLHUP | POLLRDHUP);
sqe->user_data = encode_user_data(fd, EVENT_HANGUP);
```

When keepalive times out, the kernel resets the connection. You get `POLLHUP` or `POLLRDHUP` on the CQE. Works with multishot — one SQE monitors a connection for its entire lifetime.

### Option 2: Application-Level Timeout

```c
// Multishot timeout for idle connection detection
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
struct __kernel_timespec ts = { .tv_sec = 60 };
io_uring_prep_timeout(sqe, &ts, 0, IORING_TIMEOUT_MULTISHOT | IORING_TIMEOUT_ETIME_SUCCESS);
sqe->user_data = IDLE_CHECK_TIMER;

// On each CQE: scan connection list for idle connections
// Reset idle counters when data arrives
```

### Option 3: Per-Connection Timeout (Link Chain)

```c
// Recv with linked timeout — if no data in 30s, cancel recv
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_recv(sqe, fd, buf, len, 0);
sqe->flags |= IOSQE_IO_LINK;
sqe->user_data = encode_user_data(fd, EVENT_RECV);

sqe = io_uring_get_sqe(&ring);
struct __kernel_timespec ts = { .tv_sec = 30 };
io_uring_prep_link_timeout(sqe, &ts, 0);
sqe->user_data = encode_user_data(fd, EVENT_TIMEOUT);
```

If recv completes first → timeout is cancelled. If timeout fires first → recv is cancelled with `-ECANCELED`.

## Recommendation

| Approach | Best For | Overhead |
|----------|----------|----------|
| TCP keepalive + POLL_ADD | Long-lived connections, detect network failure | 1 SQE per connection |
| Application timeout | Idle cleanup, custom timeout logic | 1 SQE + periodic scan |
| Linked recv+timeout | Request-level deadlines, short-lived connections | 2 SQEs per recv |
| Multishot timeout + scan | Many connections, coarse-grained | 1 SQE total |

For most servers: use TCP keepalive (configured at accept time) for dead connection detection, plus multishot recv for data delivery. The multishot recv returning `-ECONNRESET` or short read handles the common case. Reserve application-level timeouts for idle connection cleanup.

## Keepalive at Accept Time

Best practice: configure keepalive before the connection enters your event loop.

```c
// After multishot accept gives you a new fd:
int val = 1;
setsockopt(fd, SOL_SOCKET, SO_KEEPALIVE, &val, sizeof(val));
val = 30;  // 30 seconds idle before first probe
setsockopt(fd, IPPROTO_TCP, TCP_KEEPIDLE, &val, sizeof(val));
val = 10;  // 10 seconds between probes
setsockopt(fd, IPPROTO_TCP, TCP_KEEPINTVL, &val, sizeof(val));
val = 3;   // 3 probes before reset
setsockopt(fd, IPPROTO_TCP, TCP_KEEPCNT, &val, sizeof(val));
```

Or with URING_CMD (6.11+), submit these as linked SQEs right after accept.
