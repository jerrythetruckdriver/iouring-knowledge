# io_uring Adoption in Major Projects

Status as of March 2026. Who's using it, who isn't, and why.

## Web Servers / Proxies

### Nginx
**Status**: No native io_uring support.

Nginx uses epoll. There was a third-party proof-of-concept patch from 2020 that showed promising benchmarks for file serving (sendfile replacement), but it was never merged. The event model is deeply tied to epoll/kqueue abstractions.

**Why**: Nginx's architecture (worker process per core, epoll-based event loop) works well enough. The team's priority is stability and portability, not bleeding-edge Linux I/O. Also, Nginx is in a slow transition to Nginx Unit, which has a different architecture.

### Caddy
**Status**: No io_uring. Uses Go's net/http, which uses epoll under the hood via the Go runtime's netpoller.

**Why**: Go's runtime manages I/O scheduling. Adding io_uring would require either cgo (defeating Go's value proposition) or a pure-Go io_uring implementation. The impedance mismatch between Go's goroutine model and io_uring's completion-based model is significant.

### Envoy (proxy)
**Status**: Experimental io_uring support since 2022, but not default.

Envoy has an `IoUringSocketHandle` abstraction. It's optional and primarily targets high-throughput proxy workloads. The libevent-based event loop remains default.

**Why it's cautious**: Envoy serves production infrastructure at massive scale. Switching the core I/O path requires extensive validation. The io_uring path is for specialized deployments.

## Databases

### ScyllaDB (via Seastar)
**Status**: ✅ Production. io_uring is the preferred I/O backend.

See [seastar.md](seastar.md).

### TigerBeetle
**Status**: ✅ Production. io_uring is the *only* Linux I/O backend.

See [tigerbeetle.md](tigerbeetle.md).

### RocksDB / LevelDB
**Status**: No direct io_uring. Uses POSIX I/O or Linux AIO for async reads.

There have been patches to add io_uring as a PosixEnv backend, but nothing merged into mainline. The storage engine's I/O patterns (sequential writes, random reads) don't benefit as dramatically from io_uring as networking workloads.

### PostgreSQL
**Status**: Under discussion. No merged support.

There's been work on using io_uring for WAL writes and page reads. The challenge is PostgreSQL's process-per-connection architecture — io_uring per-process with SINGLE_ISSUER would need significant refactoring.

## Runtime / Language Support

### Rust (tokio-uring, monoio, glommio)
**Status**: ✅ Multiple production runtimes. See individual framework pages.

### Go
**Status**: Limited. [iceber/iouring-go](https://github.com/iceber/iouring-go), [pawelgaczynski/giouring](https://github.com/pawelgaczynski/giouring). No stdlib integration.

See [io-uring-go.md](io-uring-go.md).

### Java (via Project Panama / JNI)
**Status**: [netty-incubator-transport-io_uring](https://github.com/netty/netty-incubator-transport-io_uring). Netty has a functional io_uring transport.

### Node.js
**Status**: No direct support. libuv (Node's I/O layer) uses epoll. There's been discussion about io_uring in libuv but no concrete implementation.

### Python
**Status**: [liburing-cffi](https://github.com/YoSTEALTH/Liburing) bindings exist. Not mainstream.

## FUSE Implementations

### virtiofs
**Status**: ✅ Benefits directly from FUSE-over-io_uring (6.14+).

### libfuse
**Status**: Working on io_uring transport. Will benefit all FUSE filesystems built on libfuse.

## The Pattern

Adoption follows a clear pattern:
1. **New projects** (TigerBeetle, monoio) → io_uring from day one
2. **Performance-obsessed projects** (Seastar/ScyllaDB, Netty) → early adopters
3. **Established infrastructure** (Nginx, PostgreSQL) → cautious, experimental
4. **Language runtimes** (Go, Node) → blocked by architecture mismatch

The gap between groups 2 and 3 is where most of the ecosystem sits in 2026.
