# Deno Runtime and io_uring

## TL;DR

Deno uses Tokio (epoll on Linux). No io_uring. Same limitation as every Tokio-based runtime — the `AsyncRead`/`AsyncWrite` traits assume borrowed buffers.

## Architecture

```
Deno runtime
  └── deno_core (V8 + ops)
      └── tokio (async runtime)
          └── mio (cross-platform I/O)
              └── epoll (Linux) / kqueue (macOS) / IOCP (Windows)
```

Deno's I/O stack:
1. JavaScript `fetch()`, `Deno.readFile()`, etc.
2. Rust ops in `deno_core` (async functions)
3. Tokio tasks on the async runtime
4. mio → epoll for network, tokio::fs (thread pool) for file I/O

## Why No io_uring

1. **Tokio dependency** — Deno is built on Tokio. Tokio uses mio, which is readiness-based (epoll). Switching to io_uring would require either:
   - Tokio adopting io_uring (blocked by the borrowed buffer problem)
   - Deno replacing Tokio (unrealistic — too much ecosystem dependency)

2. **Cross-platform requirement** — Deno runs on Linux, macOS, Windows. io_uring is Linux-only. Deno Deploy (their cloud platform) also needs to be portable.

3. **File I/O already on thread pool** — `tokio::fs` offloads file operations to a blocking thread pool. This is "good enough" — the overhead is a thread context switch, not a syscall.

4. **Network I/O isn't the bottleneck** — For a JavaScript runtime, V8 execution, HTTP parsing, and TLS handshakes dominate. Saving syscalls on socket I/O is measurable but not transformative.

## Deno Deploy

Deno's serverless platform runs V8 isolates on edge nodes. Similar to Cloudflare Workers:
- Platform could use io_uring internally (storage caching, log shipping)
- Individual isolates don't get ring access (security)
- Currently uses tokio/epoll

## Comparison with Bun

| | Deno | Bun |
|---|------|-----|
| Language | Rust | Zig |
| Runtime | Tokio (epoll) | Custom (epoll/kqueue) |
| io_uring | No | No |
| File I/O | Thread pool | Thread pool |
| Why not io_uring | Tokio dependency | Zig async model, cross-platform |

Neither uses io_uring. Both could in theory — Bun's Zig codebase could adopt `std.Io.Evented` (which has an io_uring backend since Feb 2026), but no evidence of work in that direction.

## What Would Help

If Deno wanted io_uring (hypothetically):
- **File I/O** — Replace `tokio::fs` thread pool with io_uring reads/writes. Biggest win.
- **DNS resolution** — Replace `trust-dns`'s UDP via epoll with io_uring multishot recv.
- **fetch() internals** — hyper → h2/h1 uses tokio sockets. Would need tokio-uring integration.

None of these are planned. The Deno team's priorities are DX, standards compliance, and Deploy — not Linux-specific I/O optimization.
