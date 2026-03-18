# io_uring in Edge Compute

## The V8 Isolate Problem

Edge compute platforms (Cloudflare Workers, Deno Deploy, Fastly Compute@Edge, Vercel Edge Functions) run user code in lightweight sandboxes — typically V8 isolates or WebAssembly.

io_uring's value proposition (batched submission, completion-based I/O, zero-copy) is most impactful at the platform level, not the user code level. User code in these environments doesn't make syscalls directly.

## Architecture Layers

```
User code (JS/Wasm)  ←  can't touch io_uring
    │
V8 isolate / Wasm runtime
    │
Platform I/O layer   ←  io_uring potential here
    │
Kernel
```

The interesting question: can the platform runtime use io_uring internally to serve thousands of isolates?

## Cloudflare Workers

Cloudflare's edge runtime is built on top of their `workerd` runtime (open source, based on V8 and Cap'n Proto). The I/O layer uses `kj::AsyncIoContext` which is built on epoll.

Cloudflare investigated io_uring for their infrastructure (see [Cloudflare blog: worker pool](https://blog.cloudflare.com/missing-manuals-io_uring-worker-pool/)) but hasn't deployed it in the Workers runtime. Their concerns:

1. **Fleet heterogeneity.** Cloudflare runs kernels across a wide version range. io_uring features vary by kernel version.
2. **Security.** Workers run untrusted code. io_uring's CVE history makes it a liability in a multi-tenant environment. Cloudflare's seccomp profiles block io_uring syscalls for sandboxed processes.
3. **Not the bottleneck.** The Workers runtime is latency-bound by network hops and cache lookups, not by epoll overhead.

## Deno Deploy

Deno uses Rust's `tokio` runtime. tokio-uring exists but isn't default — tokio uses epoll via mio. Deno Deploy's edge runtime hasn't switched to io_uring. Same reasons: cross-platform requirement (they also target macOS/Windows), and the edge workload doesn't benefit much from batched I/O.

## Fastly Compute@Edge

Runs WebAssembly via Wasmtime (Cranelift). The host calls layer uses Rust async with tokio. No io_uring. Fastly's Lucet/Wasmtime sandbox model adds enough overhead that the epoll-vs-io_uring difference is noise.

## Where io_uring Helps Edge Compute

**Storage caching layers.** Edge nodes cache content on NVMe. Reading cached objects from disk is where io_uring's batched I/O, `IOPOLL`, and registered buffers shine. This is a platform-internal optimization invisible to user code.

**Connection management.** Edge nodes terminate thousands of TLS connections. Multishot accept + multishot recv on the listener sockets reduces per-connection syscall overhead. Again, platform-level.

**Log shipping.** Edge nodes generate massive telemetry. Batching log writes with linked write+fsync operations is a natural io_uring use case.

## The Wasm + io_uring Vision

Long-term, there's potential for Wasm components to submit io_uring operations through a host-mediated API:

1. Wasm component calls `wasi:io/streams#read`
2. Host translates to io_uring SQE
3. Kernel processes asynchronously
4. Host delivers CQE as Wasm stream event

This would give Wasm code io_uring's batching without exposing the raw interface. The Component Model's async support (coming in Wasm 3.0) aligns well with completion-based I/O.

But this is future work. No edge platform does this today.

## Current Reality

As of March 2026, no major edge compute platform uses io_uring in production for user-facing I/O paths. The reasons are consistent:

| Blocker | Impact |
|---------|--------|
| Kernel version requirements | Fleet can't guarantee 6.x everywhere |
| Security (CVEs, attack surface) | Multi-tenant = conservative |
| Cross-platform requirements | io_uring is Linux-only |
| Workload mismatch | Edge I/O is network-bound, not syscall-bound |
| Engineering cost | Epoll works. Switching has negative ROI. |

The first production edge use of io_uring will likely be a storage/caching optimization, not user-facing I/O.
