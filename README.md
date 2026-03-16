# io_uring Knowledge Base

Everything you need to know about Linux's async I/O interface. No fluff, no tutorials for babies. Just the technical substance.

io_uring was introduced in Linux 5.1 by Jens Axboe. It replaces the broken `aio(7)` interface and makes `epoll(7)` look like what it is: a 20-year-old design that requires 2 syscalls per event.

## Architecture

Two shared ring buffers between kernel and userspace:
- **Submission Queue (SQ)** — you write SQEs here
- **Completion Queue (CQ)** — kernel writes CQEs here

Zero-copy by design. In polling mode, zero syscalls. Memory-bound, not syscall-bound.

## Table of Contents

### [Internals](internals/)
- [Architecture](internals/architecture.md) — SQ/CQ rings, memory layout, setup flags
- [Opcodes](internals/opcodes.md) — Complete opcode reference (65+ operations)
- [Setup Flags](internals/setup-flags.md) — `io_uring_setup()` flags and what they do
- [Registration](internals/registration.md) — Buffer/file/ring registration operations
- [Features](internals/features.md) — Kernel feature flags and capability detection

### [Patterns](patterns/)
- [Networking](patterns/networking.md) — TCP accept/recv/send, multishot, zero-copy
- [File I/O](patterns/file-io.md) — Read/write, fixed buffers, registered files
- [Multishot](patterns/multishot.md) — Accept, recv, poll multishot patterns
- [Linked Operations](patterns/linked-ops.md) — SQE chaining and dependency graphs
- [Buffer Management](patterns/buffer-management.md) — Provided buffers, ring buffers, incremental consumption
- [SQ Polling](patterns/sqpoll.md) — SQPOLL mode for zero-syscall submission

### [Benchmarks](benchmarks/)
- [io_uring vs epoll](benchmarks/iouring-vs-epoll.md) — The numbers that matter
- [io_uring vs aio](benchmarks/iouring-vs-aio.md) — Why aio is dead

### [Frameworks](frameworks/)
- [Overview](frameworks/overview.md) — Who's using io_uring and how
- [TigerBeetle](frameworks/tigerbeetle.md) — Financial database, Zig + io_uring
- [Glommio](frameworks/glommio.md) — Rust thread-per-core runtime
- [Monoio](frameworks/monoio.md) — Rust thread-per-core from ByteDance
- [tokio-uring](frameworks/tokio-uring.md) — io_uring backend for Tokio
- [io_uring-go](frameworks/io-uring-go.md) — Go bindings

### [Pitfalls](pitfalls/)
- [Common Mistakes](pitfalls/common-mistakes.md) — What will bite you
- [Security](pitfalls/security.md) — Kernel attack surface and mitigations

### [Resources](resources/)
- [Links](resources/links.md) — Papers, talks, articles, source code

## Kernel Version Requirements

| Feature | Kernel |
|---------|--------|
| Basic io_uring | 5.1 |
| SQPOLL | 5.4 |
| Registered files/buffers | 5.1 |
| `io_uring_probe` | 5.6 |
| Linked SQEs | 5.3 |
| Multishot accept | 5.19 |
| Multishot recv | 6.0 |
| Send zero-copy | 6.0 |
| Provided buffer rings | 5.19 |
| `DEFER_TASKRUN` | 6.1 |
| `SINGLE_ISSUER` | 6.0 |
| `COOP_TASKRUN` | 5.19 |
| Recv zero-copy | 6.12 |
| Bind/Listen ops | 6.11 |
| Futex ops | 6.7 |
| Incremental buffer consumption | 6.10 |
| Zero-copy RX (zcrx) | 6.12 |
| NAPI busy poll | 6.9 |
| Registered wait regions | 6.13 |
| Mixed SQE/CQE sizes | 6.14 |

## Contributing

Open issues. Don't submit PRs with tutorial-level content.
