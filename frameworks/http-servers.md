# HTTP Server io_uring Adoption

Survey of major HTTP servers and their io_uring status as of March 2026.

## The Scorecard

| Server | Language | io_uring | Status | Notes |
|--------|----------|----------|--------|-------|
| **Nginx** | C | ❌ No | No plans | epoll, kqueue. Event model deeply embedded. |
| **NGINX Unit** | C | ❌ No | No plans | Async architecture, but epoll-based. Repo archived (unsupported). |
| **H2O** | C | ❌ No | Stalled | Fastly-backed. Uses its own event loop (evloop). |
| **Apache httpd** | C | ❌ No | No plans | Process/thread model. epoll for event MPM. |
| **Caddy** | Go | ❌ No | Blocked by Go runtime | Go's netpoller uses epoll. No way around it without cgo. |
| **Envoy** | C++ | ⚠️ Experimental | WIP | One ring per worker, eventfd bridge. Basic socket ops only. See [envoy.md](envoy.md). |
| **Seastar/Scylla** | C++ | ✅ Yes | Production | Thread-per-core, one ring per shard. |
| **Drogon** | C++ | ❌ No | epoll/kqueue | Uses trantor library. No io_uring backend. |
| **Actix-web** | Rust | ❌ No | tokio (epoll) | Tokio runtime, not tokio-uring. |
| **Hyper** | Rust | ❌ No | tokio (epoll) | Same story. |
| **Netty** | Java | ✅ Yes | Production (4.2) | JNI bridge. See [netty.md](netty.md). |
| **Vert.x** | Java | ✅ Via Netty | Inherited | Uses Netty transport underneath. |
| **libuv/Node.js** | C/JS | ❌ No | Under discussion | libuv has epoll. io_uring PR exists but not merged. |
| **Bun** | Zig | ❌ No | epoll | Uses custom event loop, not Zig's std.Io yet. |
| **Apache Traffic Server** | C++ | ✅ Yes | Production | Thread-local rings, SQPOLL, eventfd bridge. Disk AIO path. See [traffic-server.md](traffic-server.md). |

## Why So Few?

Three reasons:

### 1. Event Loop Lock-in

Most HTTP servers built their architecture around epoll/kqueue 10-15 years ago. The event loop is load-bearing. Ripping it out means rewriting the core.

Nginx's `ngx_epoll_module` isn't a plugin you swap. It's the heartbeat.

### 2. Portability Tax

io_uring is Linux-only. Every server that supports macOS/BSD/Windows needs a fallback path. That means maintaining two I/O backends, and the io_uring path gets less testing.

Envoy's struggle illustrates this: their `IoHandle` abstraction leaks assumptions about readiness-based I/O.

### 3. Diminishing Returns for HTTP

For a typical HTTP server doing:
- accept → recv request → process → send response → close

The syscall savings from io_uring matter less than you'd think. The bottleneck is usually application logic, TLS overhead, or upstream latency. Not the 2 extra syscalls per event that epoll costs.

Where io_uring shines for HTTP:
- **Static file serving**: splice, zero-copy send, direct I/O
- **Proxy workloads**: splice between sockets, bundle recv/send
- **High connection counts**: multishot accept, provided buffers

## Who's Actually Using It Well

**ScyllaDB's HTTP API**: Built on Seastar, naturally uses io_uring for everything including HTTP handling. Not a general-purpose HTTP server, but proves the architecture works.

**Netty-based services**: Java services using Netty 4.2's io_uring transport get real benefits on Linux. The JNI overhead is negligible compared to the syscall savings at high connection counts.

**Custom servers in Rust/Zig**: Projects building directly on io_uring (via monoio, glommio, or Zig's std.Io) can design their HTTP handling around completion-based I/O from the start. No legacy event loop to replace.

## The Pattern

General-purpose HTTP servers: unlikely to adopt io_uring. Too much rewrite risk, too little benefit for most workloads.

Specialized high-performance servers: already using it or will soon. The architecture is designed around it.

New projects: should use io_uring from day one if Linux-only is acceptable. The API is stable, the performance wins are real, and you avoid the epoll→io_uring migration tax entirely.
