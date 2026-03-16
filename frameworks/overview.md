# Framework Overview

Who's actually using io_uring in production and how they're using it.

## By Language

### Zig
- **TigerBeetle** — financial transactions database. io_uring is the I/O core. Most serious io_uring deployment in existence.

### Rust
- **Glommio** — thread-per-core runtime by Datadog (Glauber Costa). Full io_uring backend.
- **Monoio** — thread-per-core runtime from ByteDance. Production at scale.
- **tokio-uring** — io_uring backend for the Tokio ecosystem.
- **io-uring crate** — low-level Rust bindings (Tokio team).
- **Compio** — completion-based I/O framework.
- **nuclei** — proactive I/O runtime.

### Go
- **io_uring-go** (pawelgaczynski) — Go bindings via cgo.
- **uring-go** (ii64) — Pure Go io_uring bindings.
- **gaio** — gateway async I/O, optional io_uring backend.

### C/C++
- **liburing** — the reference library by Jens Axboe. If you're doing io_uring in C, use this.
- **Seastar** — ScyllaDB's framework. io_uring backend since 2020.
- **libuv** — Node.js I/O library. io_uring support added for file ops.
- **QEMU** — uses io_uring for disk I/O.

### Java
- **netty-incubator-transport-io_uring** — Netty io_uring transport.
- **Project Panama** — JDK FFI path to io_uring.

## By Use Case

| Use Case | Key Frameworks |
|----------|---------------|
| Database storage | TigerBeetle, ScyllaDB (Seastar), RocksDB |
| Network servers | Glommio, Monoio, Seastar |
| File servers | Nginx (experimental), Caddy (Go) |
| Virtual machines | QEMU, cloud-hypervisor |
| Proxies | Envoy (experimental) |

## Thread-Per-Core Model

io_uring's design naturally fits thread-per-core architectures:
- One ring per thread
- `SINGLE_ISSUER` — one submitter per ring
- `DEFER_TASKRUN` — task work runs in submitter context
- No cross-thread synchronization needed

Glommio, Monoio, and Seastar all use this model. It's the right way to build io_uring applications.

## Adoption Barriers

1. **Kernel version requirements** — production needs 5.19+ realistically
2. **Ownership semantics** — completion-based I/O requires careful buffer lifetime management
3. **Debugging difficulty** — async I/O is inherently harder to trace
4. **Security concerns** — io_uring was disabled in some container runtimes (see pitfalls/security.md)
