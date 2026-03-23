# Async DNS Resolution with io_uring

## The Problem

DNS resolution is traditionally blocking. `getaddrinfo()` blocks the calling thread, hits `/etc/nsswitch.conf`, `/etc/hosts`, potentially NSS modules, and finally sends UDP/TCP queries to resolvers. In an io_uring event loop, one blocking DNS call stalls everything.

## Approaches

### 1. io_uring with UDP Sockets (DIY DNS)

Send raw DNS queries over UDP via io_uring. Full control, maximum performance, most work.

```c
// Build DNS query packet (RFC 1035)
uint8_t query[512];
int qlen = build_dns_query(query, "example.com", DNS_TYPE_A);

// Send to resolver
struct sockaddr_in resolver = { .sin_family = AF_INET, .sin_port = htons(53) };
inet_pton(AF_INET, "8.8.8.8", &resolver.sin_addr);

io_uring_prep_sendto(sqe, udp_fd, query, qlen, 0,
                     (struct sockaddr *)&resolver, sizeof(resolver));

// Recv response (multishot for multiple queries)
io_uring_prep_recvmsg_multishot(sqe, udp_fd, &msg, 0);
sqe->flags |= IOSQE_BUFFER_SELECT;
sqe->buf_group = dns_bufs;
```

You need to handle:
- DNS packet construction/parsing
- UDP retransmission (timeout + retry)
- TCP fallback for truncated responses
- Multiple nameservers
- DNS caching

Libraries that do this: `c-ares`, `ldns`, `unbound` (libunbound).

### 2. c-ares + io_uring

c-ares is async DNS that exposes socket fds. Bridge them into io_uring:

```c
// c-ares gives you fds to watch
int fds[ARES_GETSOCK_MAXNUM];
int bitmask = ares_getsock(channel, fds, ARES_GETSOCK_MAXNUM);

// Register for poll events via io_uring
for (int i = 0; i < ARES_GETSOCK_MAXNUM; i++) {
    if (ARES_GETSOCK_READABLE(bitmask, i)) {
        io_uring_prep_poll_add(sqe, fds[i], POLLIN);
        sqe->flags |= IOSQE_IO_LINK;  // multishot
    }
}

// When io_uring reports ready, call ares_process_fd()
```

c-ares handles DNS protocol, caching, retries. io_uring handles the I/O multiplexing. Best of both worlds.

### 3. Thread Pool (Practical Default)

Use io_uring's io-wq thread pool to run blocking `getaddrinfo()`:

```c
// getaddrinfo() in io-wq worker thread
// There's no IORING_OP_GETADDRINFO, but you can:
// 1. Use IOSQE_ASYNC on a NOP to trigger io-wq, then resolve in callback
// 2. Use a separate thread pool + eventfd to signal io_uring ring
// 3. Use IORING_OP_WAITID or futex to synchronize with resolver thread
```

Not elegant, but it works. This is what most applications do — resolve once, cache, connect.

### 4. Pre-Resolved Connections

For most server workloads, DNS resolution happens at startup or configuration reload. Resolve addresses upfront, then use io_uring CONNECT with raw `sockaddr`.

```c
// At startup (blocking is fine here)
struct addrinfo *result;
getaddrinfo("backend.internal", "8080", &hints, &result);

// In event loop — no DNS, just connect
io_uring_prep_connect(sqe, fd, result->ai_addr, result->ai_addrlen);
```

## DNS over TCP with io_uring

For DNSSEC or large responses (truncated UDP), DNS falls back to TCP:

```c
// TCP DNS: 2-byte length prefix + query
io_uring_prep_connect(sqe, tcp_fd, &resolver, sizeof(resolver));
// Linked: connect → send length+query → recv length → recv response
sqe->flags |= IOSQE_IO_LINK;
io_uring_prep_send(sqe2, tcp_fd, tcp_query, tcp_qlen, 0);
sqe2->flags |= IOSQE_IO_LINK;
io_uring_prep_recv(sqe3, tcp_fd, response, sizeof(response), MSG_WAITALL);
```

## Recommendation

| Use Case | Approach |
|----------|----------|
| High-QPS DNS proxy/resolver | DIY UDP via io_uring |
| Application with many outbound connections | c-ares + io_uring poll bridge |
| Typical server | Pre-resolve + cache at startup |
| CDN/edge with per-request resolution | c-ares + io_uring or thread pool |

## What's Missing

There's no `IORING_OP_GETADDRINFO` and there probably never will be. DNS resolution involves NSS plugins, `/etc/hosts`, mDNS, LDAP — things that don't belong in the kernel. The right abstraction is async DNS libraries (c-ares) bridged to io_uring via socket fds.
