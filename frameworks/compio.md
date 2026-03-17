# Compio

Cross-platform completion-based async runtime for Rust. The only runtime that natively supports io_uring, IOCP, and polling from day one.

**Repo**: [compio-rs/compio](https://github.com/compio-rs/compio)
**License**: MIT
**Crate**: [compio](https://crates.io/crates/compio)

## Why It Exists

Every other Rust async runtime has a platform problem:

| Runtime | Linux | Windows | macOS |
|---------|-------|---------|-------|
| tokio | epoll | IOCP (via undocumented AFD) | kqueue |
| monoio | io_uring (native) | — | kqueue (fallback) |
| glommio | io_uring (native) | — | — |
| **compio** | **io_uring (native)** | **IOCP (native)** | **polling (fallback)** |

Tokio is poll-based everywhere. monoio and glommio don't do Windows. compio maps completion-based I/O natively on platforms that support it: io_uring on Linux, IOCP on Windows.

## Architecture

```
compio (facade crate)
├── compio-runtime    — executor, task spawning
├── compio-driver     — platform abstraction layer
│   ├── sys/iour/     — io_uring backend
│   ├── sys/iocp/     — Windows IOCP backend  
│   ├── sys/poll/     — poll(2) fallback
│   ├── sys/fusion/   — hybrid poll+uring
│   └── sys/stub/     — no-op (compile check)
├── compio-buf        — buffer traits (ownership-based)
├── compio-io         — AsyncRead/AsyncWrite traits
├── compio-net        — TCP, UDP, Unix sockets
├── compio-fs         — File operations
├── compio-signal     — Signal handling
└── compio-tls        — TLS integration
```

### Driver Layer

The `compio-driver` crate abstracts over platform I/O backends. Each backend implements a `Driver` trait with:
- `create_op()` — prepare an I/O operation
- `push()` — submit to the backend
- `poll()` — wait for completions
- `cancel()` — cancel in-flight ops

On Linux, the `iour` backend wraps `io-uring` (the Rust crate). Uses `COOP_TASKRUN | SINGLE_ISSUER` by default.

The `fusion` backend is interesting: hybrid poll + io_uring. Uses poll for operations io_uring doesn't support well, io_uring for everything else.

## Buffer Ownership Model

Like monoio, compio uses ownership-based I/O to solve the io_uring safety problem. Buffers must be `'static` because the kernel holds references after submission:

```rust
use compio::{fs::File, io::AsyncReadAtExt};

let file = File::open("data.bin").await?;
let (n, buf) = file.read_to_end_at(Vec::with_capacity(4096), 0).await?;
// buf is returned with data — you get ownership back after completion
```

The `(result, buffer)` return pattern. Awkward compared to tokio's `&mut [u8]`, but correct. No use-after-free, no dangling kernel references.

## io_uring Specifics

compio's io_uring backend:

- **Ring per thread**: Thread-per-core model, one ring per executor
- **Setup flags**: `COOP_TASKRUN`, `SINGLE_ISSUER` enabled by default
- **Buffer pool**: Optional `buffer_pool` module for provided buffer rings
- **Operations**: File R/W, socket accept/connect/send/recv, splice, statx, openat2
- **No SQPOLL**: Relies on submission batching instead (correct tradeoff for most workloads)
- **No multishot**: As of current implementation, doesn't use multishot accept/recv

### What It Doesn't Use (Yet)

- Multishot operations
- Registered buffers/files
- Zero-copy send/recv
- Bundle operations
- NAPI busy poll

This is the main gap vs monoio and glommio, which both leverage more advanced io_uring features.

## vs Other Runtimes

**vs tokio**: compio is completion-based, tokio is poll-based. compio can't match tokio's ecosystem, but its I/O path is fundamentally more efficient on io_uring.

**vs monoio**: Similar model (thread-per-core, ownership-based buffers). monoio uses more io_uring features (registered files, multishot). compio wins on cross-platform.

**vs glommio**: glommio is more mature for io_uring-heavy workloads (storage, databases). compio wins on portability.

## When to Use

- **Cross-platform app** that wants native completion I/O everywhere → compio
- **Linux-only server** maximizing io_uring → monoio or glommio
- **General purpose** with max ecosystem → tokio (still)

## Production Status

Active development, multiple contributors. Not at glommio or monoio maturity for io_uring-specific features, but the cross-platform story is unique. The `fusion` backend (hybrid poll + io_uring) is a pragmatic approach to feature coverage.

Good foundation. Needs more aggressive io_uring feature adoption to compete on raw Linux performance.
