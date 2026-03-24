# async-std / smol

## Status: No io_uring

async-std is **deprecated** in favor of [smol](https://github.com/smol-rs/smol). Both use `async-io` (polling-based reactor) backed by epoll on Linux.

## Architecture

```
smol
├── async-io        → epoll/kqueue/IOCP reactor
├── async-executor  → work-stealing executor
├── async-task      → task abstraction
└── blocking        → thread pool for blocking ops
```

`async-io` uses `polling` crate → `epoll_wait` on Linux. No io_uring anywhere in the stack.

## Why No io_uring

Same problem as Tokio: the `AsyncRead`/`AsyncWrite` traits use borrowed buffers. io_uring needs owned buffers (kernel holds references across submit→complete). Retrofitting completion-based I/O into a readiness-based API is an impedance mismatch.

The smol ecosystem is deliberately minimal. Adding io_uring would mean:
- New traits (owned buffer I/O)
- Platform-specific codepaths
- Significant complexity increase

## Alternatives for io_uring + Rust

| Runtime | io_uring | Model |
|---|---|---|
| **tokio-uring** | Yes | Tokio-compatible, separate runtime |
| **glommio** | Yes | Thread-per-core, owned buffers |
| **monoio** | Yes | Thread-per-core, ByteDance |
| **compio** | Yes | Cross-platform completion I/O |
| smol / async-std | No | Readiness-based (epoll) |
| tokio | No | Readiness-based (epoll) |

## The Pattern

Every readiness-based Rust runtime (Tokio, smol, async-std) faces the same barrier. The runtimes that adopted io_uring (glommio, monoio, compio) were designed for completion-based I/O from the start.

This isn't a bug — it's a design trade-off. Readiness-based APIs are simpler, portable, and good enough for most workloads. io_uring matters when you're pushing system limits.
