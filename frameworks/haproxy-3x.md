# HAProxy 3.x — Still No io_uring

## Status: No io_uring, No Plans

HAProxy 3.0 (June 2024) and 3.1 (November 2024) ship with zero io_uring code. Confirmed by source search — no references to `io_uring`, `liburing`, or `IORING` anywhere in the codebase.

## Why

HAProxy's event loop (`fd_updt`, `ev_epoll`, `ev_kqueue`, `ev_poll`, `ev_evports`) is a mature readiness-based design that's been optimized for 20+ years. The poller abstraction assumes:

1. **Readiness notification** — "fd X is readable" → call handler
2. **Non-blocking I/O** — handlers do `read()`/`write()` themselves
3. **State machine per connection** — explicit state transitions

Switching to completion-based I/O would require rewriting the connection state machine and every protocol analyzer (HTTP/1, HTTP/2, HTTP/3, TCP, QUIC).

## The Math Doesn't Work

HAProxy's bottleneck is protocol parsing (HPACK, HTTP header analysis, ACL evaluation, stick-table lookups), not syscall overhead. At HAProxy's scale:

- Each connection does ~2 syscalls per request-response cycle (read + write, or splice)
- io_uring would save ~1 syscall per cycle via batching
- That's ~100-200ns per request on modern hardware
- Protocol processing costs 1-10µs per request

Saving 2-5% of total request latency doesn't justify rewriting the event loop.

## What HAProxy Does Instead

- **splice()** for zero-copy forwarding (TCP mode)
- **SO_REUSEPORT** for multi-process load balancing
- **Thread groups** (3.0) for NUMA-aware threading
- **QUIC/HTTP3** (3.0+) as the actual performance frontier

## Comparison with Envoy

Envoy has experimental io_uring support. HAProxy doesn't. The difference: Envoy is a newer codebase with a pluggable I/O architecture (libevent). HAProxy's event loop is hand-tuned and tightly coupled to its protocol parsers.

Neither benefits meaningfully from io_uring for HTTP proxying. The syscall overhead just isn't the bottleneck at L7.

## When HAProxy Might Reconsider

If io_uring gets TLS offload integration that eliminates `SSL_read()`/`SSL_write()` overhead, or if kTLS + io_uring splice achieves true zero-copy HTTPS proxying, there might be a compelling case. But that's a kernel + OpenSSL problem, not an HAProxy problem.
