# recvmsg Ancillary Data (cmsg) Handling

## Overview

`IORING_OP_RECVMSG` supports ancillary data (control messages) via the standard `struct msghdr`. This includes `SCM_RIGHTS` (fd passing), `SCM_CREDENTIALS`, `IP_PKTINFO`, `SO_TIMESTAMP`, and any other cmsg type supported by the socket.

## SQE Layout

```c
sqe->opcode = IORING_OP_RECVMSG;
sqe->fd = sockfd;
sqe->addr = (uintptr_t)&msghdr;  // standard struct msghdr
sqe->len = 1;                     // nr_vectors (ignored for msg)
sqe->msg_flags = MSG_CMSG_CLOEXEC; // optional
```

The `msghdr.msg_control` and `msghdr.msg_controllen` fields work identically to synchronous `recvmsg(2)`.

## Multishot recvmsg + cmsg

With multishot (`IORING_RECV_MULTISHOT` in `sqe->ioprio`), ancillary data is packed into the provided buffer alongside the message. The kernel writes a `struct io_uring_recvmsg_out` header:

```c
struct io_uring_recvmsg_out {
    __u32 namelen;      // actual name length
    __u32 controllen;   // actual control data length
    __u32 payloadlen;   // actual payload length
    __u32 flags;        // msg_flags from recvmsg
};
```

Buffer layout (in provided buffer):
```
[io_uring_recvmsg_out][name (padded)][control (padded)][payload]
```

Both `name` and `control` are padded to `__alignof__(struct cmsghdr)` alignment (typically 8 bytes on 64-bit).

## Parsing Multishot cmsg

```c
struct io_uring_recvmsg_out *out = buffer;

// Skip past header + name
void *control = (char *)(out + 1) +
    CMSG_ALIGN(out->namelen);

// Walk control messages
struct cmsghdr *cmsg;
for (cmsg = (struct cmsghdr *)control;
     (char *)cmsg < (char *)control + out->controllen;
     cmsg = (struct cmsghdr *)((char *)cmsg + CMSG_ALIGN(cmsg->cmsg_len))) {
    if (cmsg->cmsg_level == SOL_SOCKET &&
        cmsg->cmsg_type == SCM_RIGHTS) {
        int *fds = (int *)CMSG_DATA(cmsg);
        int nfds = (cmsg->cmsg_len - CMSG_LEN(0)) / sizeof(int);
        // handle received file descriptors
    }
}
```

## SCM_RIGHTS (fd passing)

Works with both single-shot and multishot recvmsg. Received fds are installed into the calling process's fd table. With `MSG_CMSG_CLOEXEC`, they get `O_CLOEXEC`.

**Note:** Received fds are *not* direct descriptors. If you want them in the fixed file table, use `IORING_OP_FIXED_FD_INSTALL` after receiving.

## SO_TIMESTAMP / IP_PKTINFO

Common for UDP servers. Same cmsg parsing applies. With multishot recv, you get per-packet timestamps and source info packed in each CQE's buffer.

For hardware TX timestamps, use `SOCKET_URING_OP_TX_TIMESTAMP` via `URING_CMD` instead (6.13+).

## Buffer Sizing for cmsg

When using provided buffers with multishot recvmsg, size buffers to hold:

```
sizeof(io_uring_recvmsg_out)     // 16 bytes
+ CMSG_ALIGN(max_name_len)       // sockaddr (28 for IPv6)
+ CMSG_ALIGN(max_control_len)    // cmsg (varies)
+ max_payload_len                 // actual data
```

If the buffer is too small, the kernel truncates and sets `MSG_TRUNC`/`MSG_CTRUNC` in `out->flags`.

## Comparison

| Method | Ancillary Data | Multishot | Provided Buffers |
|--------|---------------|-----------|-----------------|
| RECVMSG | Full cmsg support | Yes | Yes (packed) |
| RECV | No cmsg | Yes | Yes (simple) |
| RECV_ZC (zcrx) | No cmsg | Yes | zcrx buffers |

Use `RECVMSG` when you need ancillary data. Use `RECV` when you don't — it's simpler and wastes less buffer space on headers.
