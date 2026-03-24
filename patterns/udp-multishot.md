# UDP Multishot Recv Patterns

Multishot recv on UDP sockets enables high-throughput datagram processing for DNS servers, QUIC endpoints, and game servers.

## Basic Setup

```c
// Multishot recvmsg on UDP socket
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_recvmsg_multishot(sqe, udp_fd, &msg, 0);
sqe->buf_group = BUF_GROUP_ID;       // provided buffer ring
sqe->flags |= IOSQE_BUFFER_SELECT;
```

Each CQE delivers one datagram with:
- `CQE_F_MORE` — multishot still active, more CQEs coming
- `CQE_F_BUFFER` — buffer ID in upper 16 bits of flags
- `io_uring_recvmsg_out` header in buffer (name_len, controllen, payloadlen, flags)

## Buffer Layout (recvmsg multishot)

```
┌──────────────────────────────────────────┐
│ io_uring_recvmsg_out header (16 bytes)   │
│   namelen, controllen, payloadlen, flags │
├──────────────────────────────────────────┤
│ Source address (struct sockaddr_in[6])    │
│   (namelen bytes, padded to alignment)   │
├──────────────────────────────────────────┤
│ Control data (cmsg)                      │
│   (controllen bytes)                     │
├──────────────────────────────────────────┤
│ Payload (the actual datagram)            │
│   (payloadlen bytes)                     │
└──────────────────────────────────────────┘
```

Parse with `io_uring_recvmsg_name()`, `io_uring_recvmsg_cmsg_firsthdr()`, `io_uring_recvmsg_payload()`.

## Buffer Sizing

UDP datagrams are bounded. Size provided buffers accordingly:

| Protocol | Max Datagram | Recommended Buffer |
|----------|-------------|-------------------|
| DNS | 512 (UDP) / 4096 (EDNS) | 4KB + header |
| QUIC | 1200-1500 (typical) | 2KB + header |
| Game | Varies, usually <1400 | 2KB + header |

Account for recvmsg_out header (16 bytes) + sockaddr (28 bytes max for IPv6) + cmsg.

Formula: `buffer_size = 16 + 28 + cmsg_space + max_payload + 8 (padding)`

## DNS Server Pattern

```c
// High-throughput DNS: multishot recv + sendmsg batching
// One SQE handles all incoming queries

// Setup: provided buffer ring with 4KB buffers
struct io_uring_buf_ring *br = setup_pbuf_ring(&ring, BUF_GROUP, 4096, 1024);

// Single multishot recvmsg
struct msghdr msg = {
    .msg_namelen = sizeof(struct sockaddr_in6),
    .msg_controllen = 0,
};
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_recvmsg_multishot(sqe, udp_fd, &msg, 0);
sqe->buf_group = BUF_GROUP;
sqe->flags |= IOSQE_BUFFER_SELECT;

// Event loop: process CQEs, parse DNS, batch responses
for (;;) {
    io_uring_submit_and_wait(&ring, 1);
    unsigned head;
    struct io_uring_cqe *cqe;
    int responses = 0;
    
    io_uring_for_each_cqe(&ring, head, cqe) {
        struct io_uring_recvmsg_out *o = io_uring_recvmsg_out(cqe);
        void *payload = io_uring_recvmsg_payload(o, &msg);
        struct sockaddr *src = io_uring_recvmsg_name(o);
        
        // Parse DNS query, build response
        dns_response_t resp = process_dns_query(payload, o->payloadlen);
        
        // Queue sendmsg response
        struct io_uring_sqe *send_sqe = io_uring_get_sqe(&ring);
        io_uring_prep_sendmsg(send_sqe, udp_fd, &resp.msg, 0);
        responses++;
        
        // Return buffer to ring
        io_uring_buf_ring_add(br, buffer, 4096, buf_id, ...);
    }
    io_uring_cq_advance(&ring, nr);
    io_uring_buf_ring_advance(br, nr);
}
```

## QUIC Server Pattern

QUIC is more complex — need to handle:
- Multiple connections on same UDP socket
- Connection ID routing
- SO_TIMESTAMP for RTT measurement

```c
// QUIC: multishot recvmsg with timestamps
struct msghdr msg = {
    .msg_namelen = sizeof(struct sockaddr_in6),
    .msg_control = cmsg_buf,
    .msg_controllen = CMSG_SPACE(sizeof(struct timespec)),  // SO_TIMESTAMPNS
};

// Enable kernel timestamps
int enable = 1;
setsockopt(udp_fd, SOL_SOCKET, SO_TIMESTAMPNS, &enable, sizeof(enable));

// Multishot recvmsg with cmsg space for timestamps
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_recvmsg_multishot(sqe, udp_fd, &msg, 0);
sqe->buf_group = BUF_GROUP;
sqe->flags |= IOSQE_BUFFER_SELECT;
```

Buffer sizing: 16 (header) + 28 (addr) + CMSG_SPACE(sizeof(timespec)) + 1500 (MTU) ≈ 1600 bytes.

## Game Server Pattern

Game servers often have many clients sending small, frequent packets:

```c
// Game server: multishot recv + NAPI busy poll for low latency
struct io_uring_napi napi = { .busy_poll_to = 100 };  // 100μs
io_uring_register_napi(&ring, &napi);

// Multishot recvmsg
// Small buffers (game packets usually <256 bytes)
struct io_uring_buf_ring *br = setup_pbuf_ring(&ring, BUF_GROUP, 512, 4096);

// Combine with registered wait for tight event loop
struct io_uring_reg_wait rw = {
    .min_wait_usec = 500,  // batch up to 500μs
    .flags = IORING_REG_WAIT_TS,
};
```

## Multishot Termination

Multishot recvmsg terminates when:
- CQE without `CQE_F_MORE` (kernel terminated it)
- Explicit `ASYNC_CANCEL` by application
- Buffer exhaustion (no buffers in group)
- Socket error

On termination, resubmit the multishot SQE.

## Performance vs recvmmsg()

| Aspect | recvmmsg() | io_uring multishot |
|--------|-----------|-------------------|
| Syscalls | 1 per batch | 0 (after setup) |
| Batching | Explicit (vlen param) | Implicit (CQ batching) |
| Buffer management | Pre-allocated array | Provided buffer ring |
| Max batch | vlen (typically 64-256) | CQ size |
| Latency floor | Syscall overhead | io_uring overhead |
| Kernel version | 2.6.33 | 5.19 (multishot recv) |

For high-throughput UDP, multishot recvmsg with NAPI busy poll approaches the syscall efficiency of recvmmsg while being more naturally async.

## Gotchas

1. **MSG_TRUNC**: If datagram exceeds buffer size, `io_uring_recvmsg_out.flags` will have MSG_TRUNC. The datagram is silently truncated.
2. **Buffer refill**: Must return buffers to provided buffer ring promptly. Starvation terminates the multishot.
3. **Source address alignment**: `io_uring_recvmsg_name()` may not be naturally aligned. Copy before casting to sockaddr_in6 on strict-alignment architectures.
4. **Incremental consumption**: Use `IOU_PBUF_RING_INC` (6.10) to partially consume buffers — useful if datagrams are much smaller than buffer size.
