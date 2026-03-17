# Production io_uring: Cloudflare & Fastly

## Cloudflare

### Current Status

Cloudflare has been investigating and documenting io_uring extensively but adoption in their edge proxy is **incremental, not wholesale**.

### Known Usage

- **Research & documentation**: Cloudflare authored the definitive blog post on the io_uring worker pool ("Missing Manuals" series, Feb 2022). This level of investigation implies active evaluation for production workloads.
- **Edge proxy (Pingora)**: Written in Rust, uses tokio as the async runtime. tokio itself uses epoll, not io_uring. Pingora doesn't directly use io_uring as of known public information.
- **Storage/caching**: Potential use for disk I/O in their cache layer, where io_uring's storage I/O advantages (O_DIRECT, registered buffers, IOPOLL) are most compelling.

### Cloudflare's Worker Pool Investigation

Key findings from their blog post:
- io_uring creates one worker thread per IOSQE_ASYNC request for unbounded work
- Bounded workers (file/block I/O) limited by SQ ring size
- Unbounded workers (socket I/O) limited by RLIMIT_NPROC
- Worker creation is per-NUMA-node
- `IORING_REGISTER_IOWQ_MAX_WORKERS` caps the pool

This investigation revealed that io_uring's thread pool behavior is under-documented — a common theme that slows production adoption.

### Why Not Full Adoption

- Cloudflare's edge runs on many kernel versions across their fleet; io_uring features are kernel-version-dependent
- Security concerns: io_uring has been a significant source of kernel CVEs (60% of kCTF exploits)
- Their proxy workload is primarily network I/O, where epoll is battle-tested and io_uring's advantages are smaller than for storage I/O

## Fastly

### Current Status

Limited public information about io_uring usage in production.

### Known Context

- **Lucet/Wasmtime**: Fastly's compute platform runs Wasm workloads. The runtime's I/O is abstracted through their platform layer.
- **H2O**: Fastly contributes to the H2O HTTP server, which uses its own event loop (primarily epoll-based on Linux).
- **Potential**: Fastly's use case (CDN edge, high-throughput proxying) is similar to Cloudflare's. Same adoption barriers apply.

## The CDN Adoption Pattern

CDN/edge providers share common characteristics that affect io_uring adoption:

1. **Network I/O dominant**: Most work is TCP proxy (epoll is fine)
2. **Fleet heterogeneity**: Multiple kernel versions in production
3. **Security surface**: io_uring CVEs are a real concern at scale
4. **Incremental approach**: Storage I/O (cache) first, networking later

The trajectory: io_uring enters through the storage path (cache reads, log writes) where the benefit over epoll is clear. Network I/O adoption follows once kernel versions stabilize and the security story improves.

## What Production Adoption Looks Like

Companies actually running io_uring in production for core workloads:

| Company | Use Case | Why |
|---------|----------|-----|
| ScyllaDB | Database I/O | Thread-per-core, needs async disk I/O |
| Datadog | Agent (Glommio) | Thread-per-core Rust services |
| ByteDance | Various (Monoio) | Internal Rust async runtime |
| Redpanda | Streaming storage | Seastar-based, async disk I/O |
| Samsung | Storage stack | TigerBeetle-style direct ring usage |

The pattern: companies building **storage-intensive, latency-sensitive** systems adopt io_uring first. Pure network proxies are slower to switch because epoll works well enough.
