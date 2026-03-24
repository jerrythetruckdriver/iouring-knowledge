# Unix Domain Sockets + io_uring

## What Works

All standard socket ops work on AF_UNIX:

| Operation | Opcode | Notes |
|-----------|--------|-------|
| SOCKET | `IORING_OP_SOCKET` | AF_UNIX, SOCK_STREAM/DGRAM/SEQPACKET |
| BIND | `IORING_OP_BIND` (6.14) | Bind to path or abstract namespace |
| LISTEN | `IORING_OP_LISTEN` (6.14) | |
| ACCEPT | `IORING_OP_ACCEPT` | Multishot works |
| CONNECT | `IORING_OP_CONNECT` | |
| SEND/RECV | `IORING_OP_SEND/RECV` | Multishot recv works |
| SENDMSG/RECVMSG | `IORING_OP_SENDMSG/RECVMSG` | For ancillary data |
| SHUTDOWN | `IORING_OP_SHUTDOWN` | |
| CLOSE | `IORING_OP_CLOSE` | |

## SCM_RIGHTS: Passing File Descriptors

The main reason to use AF_UNIX over TCP. Works via sendmsg/recvmsg with cmsg.

### Sending FDs

```c
// Build msghdr with SCM_RIGHTS cmsg
struct msghdr msg = {};
struct cmsghdr *cmsg;
char buf[CMSG_SPACE(sizeof(int))];
int fd_to_send = open("/tmp/file", O_RDONLY);

msg.msg_control = buf;
msg.msg_controllen = sizeof(buf);
cmsg = CMSG_FIRSTHDR(&msg);
cmsg->cmsg_level = SOL_SOCKET;
cmsg->cmsg_type = SCM_RIGHTS;
cmsg->cmsg_len = CMSG_LEN(sizeof(int));
memcpy(CMSG_DATA(cmsg), &fd_to_send, sizeof(int));

// iovec with at least 1 byte (required)
struct iovec iov = { .iov_base = "x", .iov_len = 1 };
msg.msg_iov = &iov;
msg.msg_iovlen = 1;

io_uring_prep_sendmsg(sqe, unix_fd, &msg, 0);
```

### Receiving FDs

```c
struct msghdr msg = {};
char ctrl_buf[CMSG_SPACE(sizeof(int) * MAX_FDS)];
char data_buf[64];
struct iovec iov = { .iov_base = data_buf, .iov_len = sizeof(data_buf) };

msg.msg_iov = &iov;
msg.msg_iovlen = 1;
msg.msg_control = ctrl_buf;
msg.msg_controllen = sizeof(ctrl_buf);

io_uring_prep_recvmsg(sqe, unix_fd, &msg, 0);
// After CQE: parse cmsg for SCM_RIGHTS, extract fds
```

### With Multishot RECVMSG

Multishot recvmsg works for AF_UNIX but cmsg parsing requires `io_uring_recvmsg_out` header navigation. Buffer must be sized for cmsg overhead:

```
buffer_size >= sizeof(io_uring_recvmsg_out) + namelen + CMSG_SPACE(n * sizeof(int)) + payload
```

## SCM_CREDENTIALS

Pass process credentials (pid, uid, gid) over Unix sockets. Same sendmsg/recvmsg pattern, `cmsg_type = SCM_CREDENTIALS`, data is `struct ucred`.

Requires `SO_PASSCRED` on receiving socket — set via `SOCKET_URING_OP_SETSOCKOPT` (6.11) or synchronous setsockopt before entering io_uring.

## Abstract Namespace

Linux-specific. Bind to `\0name` instead of filesystem path. No filesystem cleanup needed. Works identically with io_uring.

## MSG_RING for Cross-Process IPC

If both processes have io_uring rings, `IORING_OP_MSG_RING` with `IORING_MSG_SEND_FD` can pass fixed descriptors between rings — no Unix socket needed. But requires ring fd access (typically via SCM_RIGHTS first, or shared memory).

## Pattern: Process Pool with FD Passing

```
Parent:
  ACCEPT (multishot) on TCP socket
  → on new connection:
    SENDMSG to least-busy worker (SCM_RIGHTS: connection fd)

Worker:
  RECVMSG on Unix socket (multishot)
  → extract fd from cmsg
  → RECV/SEND on connection fd
```

## Gotchas

- **sendmsg/recvmsg required** for ancillary data — plain send/recv can't carry cmsg
- **Multishot + cmsg**: buffer sizing must account for control message overhead
- **SCM_RIGHTS delivery is all-or-nothing**: if cmsg doesn't fit in buffer, fds are silently dropped
- **Received fds are new descriptors**: they consume fd table slots. Close them or use FIXED_FD_INSTALL to put them in the fixed file table
- **SOCK_DGRAM preserves message boundaries**, SOCK_STREAM doesn't — relevant for cmsg alignment
