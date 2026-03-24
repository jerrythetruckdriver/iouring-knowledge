# sendmsg_zc Patterns (MSG_ZEROCOPY for UDP)

`IORING_OP_SENDMSG_ZC` (6.1) extends zero-copy send to UDP and complex message types.

## How It Works

```c
struct msghdr msg = {
    .msg_name = &dest_addr,
    .msg_namelen = sizeof(dest_addr),
    .msg_iov = iovs,
    .msg_iovlen = nr_iovs,
};

struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_sendmsg_zc(sqe, sockfd, &msg, 0);
```

Two CQEs per operation:
1. **Send CQE**: bytes sent (or error). `CQE_F_MORE` set.
2. **Notification CQE**: `CQE_F_NOTIF` set. Buffer safe to reuse.

## UDP Zero-Copy

UDP is the primary use case for `sendmsg_zc`. TCP has `IORING_OP_SEND_ZC` which is simpler.

```c
// DNS response: zero-copy UDP send
struct sockaddr_in6 client;
struct iovec iov = { .iov_base = dns_response, .iov_len = resp_len };
struct msghdr msg = {
    .msg_name = &client,
    .msg_namelen = sizeof(client),
    .msg_iov = &iov,
    .msg_iovlen = 1,
};

io_uring_prep_sendmsg_zc(sqe, udp_fd, &msg, 0);
```

## Vectored Zero-Copy Send

Combine headers and payloads without copying:

```c
struct iovec iovs[3] = {
    { .iov_base = header,  .iov_len = 40 },
    { .iov_base = payload, .iov_len = 1400 },
    { .iov_base = trailer, .iov_len = 8 },
};
struct msghdr msg = { .msg_iov = iovs, .msg_iovlen = 3 };
io_uring_prep_sendmsg_zc(sqe, fd, &msg, 0);
```

## Ancillary Data (cmsg)

`sendmsg_zc` supports cmsg for:
- **`IP_PKTINFO`**: source address selection
- **`IPV6_PKTINFO`**: source address for IPv6
- **`UDP_SEGMENT`** (GSO): segment large payloads into MTU-sized packets
- **`SO_TXTIME`**: transmit time scheduling

```c
// GSO: send 64KB as 1500-byte segments in one syscall
char cmsg_buf[CMSG_SPACE(sizeof(uint16_t))];
struct msghdr msg = { ... };
msg.msg_control = cmsg_buf;
msg.msg_controllen = sizeof(cmsg_buf);

struct cmsghdr *cmsg = CMSG_FIRSTHDR(&msg);
cmsg->cmsg_level = SOL_UDP;
cmsg->cmsg_type = UDP_SEGMENT;
cmsg->cmsg_len = CMSG_LEN(sizeof(uint16_t));
*(uint16_t *)CMSG_DATA(cmsg) = 1472; // segment size

io_uring_prep_sendmsg_zc(sqe, fd, &msg, 0);
```

## Notification Handling

```c
// Must consume both CQEs before reusing buffer
io_uring_cqe *cqe;
io_uring_wait_cqe(&ring, &cqe);

if (cqe->flags & IORING_CQE_F_MORE) {
    // Send complete, notification pending
    io_uring_cqe_seen(&ring, cqe);

    // Wait for notification
    io_uring_wait_cqe(&ring, &cqe);
    assert(cqe->flags & IORING_CQE_F_NOTIF);
    // NOW safe to reuse/free buffer
}
io_uring_cqe_seen(&ring, cqe);
```

## When to Use sendmsg_zc

| Use Case | Recommendation |
|---|---|
| UDP responses (DNS, QUIC) | Yes — avoids copy for every packet |
| Large UDP payloads with GSO | Yes — GSO + zero-copy = maximum throughput |
| TCP send | Use `IORING_OP_SEND_ZC` instead (simpler) |
| Small packets (<256 bytes) | No — copy is cheaper than notification overhead |
| Multicast/broadcast | Yes — same buffer sent to multiple destinations |

## send_zc vs sendmsg_zc

| Feature | `SEND_ZC` | `SENDMSG_ZC` |
|---|---|---|
| Protocol | TCP (connected) | TCP, UDP, any |
| Addressing | Implicit (connected fd) | Explicit (msg_name) |
| Vectored I/O | No (single buffer) | Yes (iovec array) |
| cmsg support | No | Yes (GSO, pktinfo, txtime) |
| Notification | Same (CQE_F_NOTIF) | Same |
| Complexity | Lower | Higher |

For TCP, prefer `SEND_ZC`. For UDP or anything needing cmsg/vectored I/O, use `sendmsg_zc`.
