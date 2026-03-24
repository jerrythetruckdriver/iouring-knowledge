# PROXY Protocol and io_uring

## What's PROXY Protocol?

HAProxy's PROXY protocol (v1 text, v2 binary) prepends a header to TCP connections with the original client IP/port. Used behind load balancers.

## The Pattern with io_uring

PROXY protocol is a one-time header at connection start. It doesn't affect io_uring architecture — just the first recv on a new connection.

### Multishot Accept + PROXY Header Recv

```c
// 1. Multishot accept
sqe = get_sqe();
io_uring_prep_multishot_accept(sqe, listen_fd, NULL, NULL, 0);

// 2. On each accept CQE, start a multishot recv
sqe = get_sqe();
io_uring_prep_recv(sqe, client_fd, NULL, 0, 0);
sqe->ioprio |= IORING_RECV_MULTISHOT;
sqe->flags |= IOSQE_BUFFER_SELECT;
sqe->buf_group = PROXY_BUF_GROUP;

// 3. First CQE contains the PROXY header — parse it
// 4. Subsequent CQEs contain actual application data
```

### PROXY v2 Header (Binary)

```c
// 12-byte signature + 4-byte header + variable address block
// Total: 16 + address length (max ~216 bytes for IPv6+TLVs)
#define PROXY_V2_MIN_LEN 16
#define PROXY_V2_MAX_LEN 536  // 16 + 520 max address block

// Parse from first recv buffer:
struct proxy_v2_header {
    uint8_t sig[12];    // \x0D\x0A\x0D\x0A\x00\x0D\x0A\x51\x55\x49\x54\x0A
    uint8_t ver_cmd;    // version (high nibble) | command (low nibble)
    uint8_t fam;        // address family | transport
    uint16_t len;       // address block length (network byte order)
};
```

### PROXY v1 Header (Text)

```
PROXY TCP4 192.168.1.1 10.0.0.1 56789 80\r\n
```

Text-based, terminated by `\r\n`. Max 107 bytes. Parse from first recv buffer, find `\r\n`, everything after is application data.

## Provided Buffer Sizing

With provided buffer rings, size buffers to handle PROXY header + initial application data:

```c
// Buffer must hold at least PROXY header + some app data
// PROXY v2: up to 536 bytes header
// Typical: 256 bytes is enough for most PROXY v2 headers
#define BUF_SIZE (536 + 4096)  // header + initial payload
```

With incremental consumption (`IOU_PBUF_RING_INC`), you can use larger buffers — the PROXY header bytes are consumed, and the remaining buffer is available for application data.

## Performance Impact

PROXY protocol adds ~0 overhead to io_uring. It's a fixed-cost parse on the first recv of each connection. The io_uring patterns (multishot recv, provided buffers, zero-copy) all work normally after the header is consumed.

If anything, io_uring is better for PROXY protocol than epoll because:
1. Accept + first recv can be a linked chain (one submission)
2. Multishot recv delivers the PROXY header and subsequent data with the same SQE
3. No extra syscall for the header read
