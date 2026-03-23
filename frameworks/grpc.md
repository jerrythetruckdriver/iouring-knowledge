# gRPC — io_uring Status

## Status: Not Adopted

gRPC C core uses its own event engine abstraction. The default Linux backend is epoll (via `PosixEventEngine`). There are exactly two GitHub issues mentioning io_uring (#27097, #41557) — one was closed as "not actionable" by a maintainer who stated "I have no idea what io_uring is."

## Why gRPC Doesn't Use io_uring

1. **Event engine abstraction** — gRPC's `EventEngine` interface is designed around readiness notification (epoll/kqueue/IOCP). Switching to completion-based I/O would require rearchitecting the interface.

2. **HTTP/2 framing overhead** — gRPC runs over HTTP/2. The bottleneck is protocol processing (header compression, stream multiplexing, flow control), not raw I/O. Reducing syscall overhead via io_uring doesn't help when you're spending cycles on HPACK encoding.

3. **Cross-platform requirement** — gRPC supports Linux, macOS, Windows, Android, iOS. Linux-only optimizations have high maintenance cost relative to benefit.

4. **Protobuf serialization** — The CPU cost of serializing/deserializing Protocol Buffers often dominates I/O wait time, especially for small messages.

5. **Language runtimes** — gRPC-Go uses Go's netpoller (epoll), gRPC-Java uses Netty (which *does* have io_uring transport), gRPC-Python wraps the C core. Only Java gets close.

## gRPC-Java via Netty

The one path to io_uring in gRPC: Netty 4.2's `io_uring` transport. If you use gRPC-Java with Netty transport and configure `IoUringEventLoopGroup` instead of `EpollEventLoopGroup`, you get io_uring for socket I/O.

This is a Netty feature, not a gRPC feature. gRPC just happens to run on Netty.

## Where io_uring Would Actually Help gRPC

- **Proxy/load balancer** — Envoy (gRPC's recommended proxy) has experimental io_uring. Batch accept + multishot recv for high connection counts.
- **File-backed streaming** — Large file transfers via gRPC streaming could use io_uring for disk reads. Niche use case.
- **Connection establishment** — Batch connect() for client-side load balancing across many backends.

## Bottom Line

gRPC's bottleneck is protocol processing, not syscall overhead. io_uring would shave microseconds off I/O that's dominated by milliseconds of HTTP/2 + protobuf work. Not worth the engineering investment for the C core team.

If you need io_uring + RPC, look at Cap'n Proto (uses kj event loop, could integrate), or build a custom protocol on raw io_uring sockets. HTTP/2 framing overhead might be your actual problem.
