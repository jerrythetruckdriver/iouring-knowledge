# Multicast UDP Patterns

## Overview

Multicast UDP with io_uring works through standard socket operations. No special opcodes. The io_uring advantage: batched multishot recv for high-throughput group traffic.

## Setup

Multicast group join is a control-plane operation — do it synchronously before submitting I/O:

```c
int fd = socket(AF_INET, SOCK_DGRAM, 0);

// Allow address reuse
int reuse = 1;
setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(reuse));

// Bind to multicast port
struct sockaddr_in addr = {
    .sin_family = AF_INET,
    .sin_port = htons(5000),
    .sin_addr.s_addr = INADDR_ANY,
};
bind(fd, (struct sockaddr *)&addr, sizeof(addr));

// Join multicast group
struct ip_mreq mreq = {
    .imr_multiaddr.s_addr = inet_addr("239.1.1.1"),
    .imr_interface.s_addr = INADDR_ANY,
};
setsockopt(fd, IPPROTO_IP, IP_ADD_MEMBERSHIP, &mreq, sizeof(mreq));
```

Or via URING_CMD (6.11+) for async SETSOCKOPT — but group join is a one-time operation, not worth the complexity.

## Receiving Multicast Traffic

### Multishot RECVMSG (Recommended)

```c
// Provided buffer ring for zero-alloc recv
struct io_uring_buf_ring *br = setup_buf_ring(&ring, BUF_GROUP, 256, 2048);

struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
struct msghdr msg = {
    .msg_namelen = sizeof(struct sockaddr_in),  // capture sender address
};
io_uring_prep_recvmsg_multishot(sqe, fd, &msg, 0);
sqe->buf_group = BUF_GROUP;
sqe->flags |= IOSQE_BUFFER_SELECT;
sqe->user_data = MCAST_RECV;
```

One SQE, unlimited CQEs. Each CQE contains:
- `io_uring_recvmsg_out` header (namelen, controllen, payloadlen, flags)
- Sender address (who sent to the group)
- Payload

### Buffer Sizing

Multicast datagrams have known MTU constraints:
- Ethernet: ≤1472 bytes (1500 - IP - UDP headers)
- Jumbo frames: ≤8972 bytes
- Common multicast protocols: 512-1400 bytes

Provided buffer size = max datagram size + `sizeof(struct io_uring_recvmsg_out)` + `sizeof(struct sockaddr_in)`.

For 1500 MTU: 1536 bytes per buffer is safe.

## Sending to Multicast Groups

```c
struct sockaddr_in dest = {
    .sin_family = AF_INET,
    .sin_port = htons(5000),
    .sin_addr.s_addr = inet_addr("239.1.1.1"),
};

struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_sendto(sqe, fd, data, len, 0,
                     (struct sockaddr *)&dest, sizeof(dest));
```

For high-throughput multicast send, use bundle send (6.10+) to batch multiple datagrams.

## Multicast + NAPI Busy Poll

For latency-sensitive multicast (market data, telemetry):

```c
struct io_uring_napi n = {
    .busy_poll_to = 100,  // 100μs busy poll timeout
};
io_uring_register_napi(&ring, &n);
```

Requires:
- NIC RSS steering multicast traffic to the right queue
- ethtool ring buffer tuning
- IRQ affinity matching

## Use Cases

| Application | Pattern | Buffer Size |
|-------------|---------|-------------|
| Market data feed | Multishot recvmsg + NAPI | 1024-1400B |
| Video multicast | Multishot recvmsg, large buffers | 1400B (TS packets) |
| Service discovery | Single recv, low rate | 512B |
| Sensor telemetry | Multishot recvmsg + timestamp cmsg | 256-512B |
| Game state sync | Multishot recvmsg + NAPI | 128-512B |

## Source-Specific Multicast (SSM)

```c
struct ip_mreq_source mreqs = {
    .imr_multiaddr.s_addr = inet_addr("232.1.1.1"),
    .imr_sourceaddr.s_addr = inet_addr("10.0.0.1"),
    .imr_interface.s_addr = INADDR_ANY,
};
setsockopt(fd, IPPROTO_IP, IP_ADD_SOURCE_MEMBERSHIP, &mreqs, sizeof(mreqs));
```

SSM groups (232.0.0.0/8) filter at the kernel/NIC level. Combined with NAPI busy poll and multishot recv, this is the lowest-latency path for receiving from a known source.

## What Doesn't Work

- **Multicast loop control**: `IP_MULTICAST_LOOP` must be set via setsockopt (or URING_CMD SETSOCKOPT)
- **TTL setting**: `IP_MULTICAST_TTL` — same, control-plane setsockopt
- **Interface selection**: `IP_MULTICAST_IF` — control-plane
- **Group management**: join/leave are synchronous operations

io_uring handles the data plane (send/recv). Group management stays synchronous.
