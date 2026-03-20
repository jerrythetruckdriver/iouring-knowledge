# Rust Web Frameworks and io_uring

## The Short Answer

No mainstream Rust web framework uses io_uring for networking. They all sit on Tokio, which uses epoll.

## Framework Survey

### Actix-web

- **Runtime**: Tokio (epoll via mio)
- **io_uring**: No
- **Architecture**: Multi-threaded Tokio runtime, one acceptor thread, N worker threads
- **Why not**: Actix moved from its own actor runtime to Tokio in v4. Tokio's networking is epoll-based. No plans to change.
- **Could it?**: Only if Tokio's networking layer switched to io_uring. See tokio-uring below.

### Axum

- **Runtime**: Tokio + Hyper (epoll via mio)
- **io_uring**: No
- **Architecture**: Thin routing layer on top of Hyper. Uses `tokio::net::TcpListener` directly.
- **Why not**: Axum is explicitly a Tokio ecosystem library. Zero custom I/O.
- **Could it?**: Same dependency on Tokio networking as Actix.

### Warp

- **Runtime**: Tokio + Hyper (epoll via mio)
- **io_uring**: No
- **Note**: Maintenance mode. Same Tokio dependency.

### Rocket

- **Runtime**: Tokio (since v0.5)
- **io_uring**: No
- **Note**: Moved from its own runtime to Tokio. Same story.

### Poem

- **Runtime**: Tokio + Hyper
- **io_uring**: No

## The Tokio Problem

Every framework above depends on Tokio for async I/O. Tokio uses `mio`, which uses epoll on Linux. The io_uring question is really a Tokio question.

### tokio-uring

Exists as a separate crate, **not** integrated into mainline Tokio. Key issues:

1. **Buffer ownership**: io_uring needs to own buffers during I/O. Tokio's `AsyncRead`/`AsyncWrite` traits use `&[u8]`/`&mut [u8]` borrowed references. Fundamentally incompatible.
2. **Separate runtime**: `tokio-uring` is a separate runtime, not a drop-in backend for Tokio. You can't use `axum` with `tokio-uring` without rewriting the I/O layer.
3. **Single-threaded**: `tokio-uring` runs on a single-threaded Tokio runtime (SINGLE_ISSUER constraint). Multi-threaded requires one ring per thread.

### Why Tokio Won't Just Switch

- `AsyncRead`/`AsyncWrite` trait design assumes borrowed buffers
- io_uring requires owned/pinned buffers (completion-based model)
- Changing this breaks the entire Tokio ecosystem
- Performance gains for HTTP workloads are marginal (HTTP is request/response, not bulk I/O)

## Frameworks That Could Use io_uring

### Thread-per-core Rust frameworks

These aren't "web frameworks" in the Actix/Axum sense, but they're the ones actually using io_uring:

| Framework | io_uring | Model |
|---|---|---|
| **Glommio** | Yes | Thread-per-core runtime (Datadog) |
| **Monoio** | Yes | Thread-per-core runtime (ByteDance) |
| **Compio** | Yes | Cross-platform completion runtime |

Building an HTTP server on Glommio or Monoio gives you io_uring networking. But you lose the Tokio ecosystem (tower middleware, hyper, tonic, etc).

## The Real Bottleneck

For HTTP servers, the bottleneck is rarely the event loop syscall overhead. It's:
- TLS handshakes (CPU-bound)
- Serialization/deserialization (CPU-bound)
- Database queries (external latency)
- Application logic

epoll adds ~2 syscalls per event vs io_uring's batched submission. At 100K req/s, that's ~200K extra syscalls. On modern hardware, that's noise. The win matters at 1M+ req/s or for specific patterns (bulk file serving, proxy workloads).

## What Would Change This

1. **Rust async trait reform**: If the async ecosystem moved to completion-based I/O traits (owned buffers), io_uring could be a transparent backend
2. **Tokio io_uring backend**: If Tokio added io_uring as an alternative to mio, all frameworks benefit automatically
3. **Neither is imminent**: The Tokio team has discussed it but the trait breakage is too severe

## Recommendation

If you need io_uring in Rust networking:
- Use **monoio** or **glommio** directly and build your HTTP handling on top
- Accept that you're outside the Tokio ecosystem
- This makes sense for **high-throughput proxies**, **storage servers**, and **game servers** — not typical web APIs
