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
- [Kernel Changelog](internals/kernel-changelog.md) — Per-release io_uring changes (6.14–6.19) ✨
- [liburing API](internals/liburing-api.md) — liburing userspace API reference ✨
- [Worker Pool](internals/worker-pool.md) — io-wq thread pool internals, limits, NUMA ✨✨
- [Memory Ordering](internals/memory-ordering.md) — Shared memory barriers, SQ/CQ synchronization ✨✨
- [Registered Wait](internals/registered-wait.md) — io_uring_reg_wait regions (6.13+) ✨✨
- [NO_SQARRAY & SQ_REWIND](internals/no-sqarray.md) — Removing SQ indirection, flat batch submission ✨✨✨
- [Mixed-Size SQEs/CQEs](internals/mixed-size.md) — CQE_MIXED and SQE_MIXED modes ✨✨✨
- [Query API](internals/query-api.md) — IORING_REGISTER_QUERY unified capability detection ✨✨✨
- [Hybrid IOPOLL](internals/hybrid-iopoll.md) — Adaptive polling with sleep delay (6.13+) ✨⁴
- [Memory Regions](internals/mem-region.md) — IORING_REGISTER_MEM_REGION, pre-mapped shared memory ✨⁴
- [NVMe Passthrough](internals/nvme-passthrough.md) — URING_CMD for direct NVMe command submission ✨⁴
- [NO_IOWAIT](internals/no-iowait.md) — Suppress iowait accounting (6.14+) ✨⁴
- [Ring Resizing](internals/ring-resizing.md) — Runtime SQ/CQ ring resize (6.13+) ✨⁴
- [Kernel Changelog 6.15–6.17](internals/kernel-changelog-6.15-6.17.md) — zcrx, epoll_wait, DMA-BUF, pipe ✨⁵
- [eBPF Integration](internals/ebpf-integration.md) — BPF filtering, iterators, async convergence ✨⁶
- [liburing Releases](internals/liburing-releases.md) — Release history, 2.14 current, API changes ✨⁶
- [DMA-BUF Zero-Copy RX](internals/dmabuf-zcrx.md) — GPU/NPU direct networking via DMA-BUF (6.16) ✨⁶
- [Clock Registration](internals/clock-register.md) — IORING_REGISTER_CLOCK custom clock source ✨⁷
- [Mixed-Size SQEs/CQEs (Detail)](internals/mixed-sqe-cqe.md) — CQE_F_SKIP, CQE_F_32, memory savings math ✨⁷
- [Task Restrictions](internals/task-restrictions.md) — Per-task opcode/flag limits, ring sandboxing ✨⁷
- [Kernel Changelog 6.18](internals/kernel-changelog-6.18.md) — Mixed CQEs, multishot URING_CMD, query API ✨⁷
- [Request Lifecycle](internals/request-lifecycle.md) — SQE→CQE full kernel execution path ✨⁸
- [NUMA](internals/numa.md) — Ring placement, worker affinity, cross-node pitfalls ✨⁸
- [PBUF_STATUS](internals/pbuf-status.md) — Buffer group introspection (opcode 26) ✨⁸
- [Embedded & Real-Time](internals/embedded-realtime.md) — PREEMPT_RT, deadline scheduling, embedded Linux ✨⁸
- [NVMe Admin Commands](internals/nvme-admin.md) — Admin queue via URING_CMD, identify, SMART, features ✨⁹
- [Kernel Changelog 6.19](internals/kernel-changelog-6.19.md) — Mixed SQEs, zcrx ctrl, getsockname, queries ✨¹³
- [eBPF Iterators](internals/ebpf-iterators.md) — BPF iter introspection: task, fd, tracepoint approaches ✨¹⁴
- [READV_FIXED / WRITEV_FIXED](internals/readv-writev-fixed.md) — Vectored I/O with registered buffers (6.15) ✨¹⁴
- [GPIO/SPI/I2C](internals/gpio-spi-i2c.md) — Char device URING_CMD potential: status and workarounds ✨¹⁴

### [Patterns](patterns/)
- [Networking](patterns/networking.md) — TCP accept/recv/send, multishot, zero-copy
- [File I/O](patterns/file-io.md) — Read/write, fixed buffers, registered files
- [Multishot](patterns/multishot.md) — Accept, recv, poll multishot patterns
- [Linked Operations](patterns/linked-ops.md) — SQE chaining and dependency graphs
- [Buffer Management](patterns/buffer-management.md) — Provided buffers, ring buffers, incremental consumption
- [SQ Polling](patterns/sqpoll.md) — SQPOLL mode for zero-syscall submission
- [Zero-Copy RX](patterns/zero-copy-rx.md) — zcrx deep dive with structs and flow ✨
- [epoll Migration](patterns/epoll-migration.md) — Incremental migration from epoll to io_uring ✨
- [Error Handling](patterns/error-handling.md) — CQE errors, EINTR, link chains, multishot ✨✨
- [Bundle Operations](patterns/bundle-ops.md) — Batched buffer recv/send, vectorized send ✨✨✨
- [Fixed File Management](patterns/fixed-files.md) — Registration, auto-alloc, lifecycle patterns ✨✨✨
- [Socket Options](patterns/socket-options.md) — Async getsockopt/setsockopt via URING_CMD ✨✨✨
- [R/W Attributes](patterns/rw-attributes.md) — PI metadata for data integrity verification ✨⁴
- [Futex Operations](patterns/futex-ops.md) — Async futex wait/wake/waitv (6.7+) ✨⁵
- [Clone Buffers](patterns/clone-buffers.md) — Share registered buffer tables between rings ✨⁵
- [Send MSG_RING Without Ring](patterns/send-msg-ring.md) — Signal a ring without owning one ✨⁵
- [Waitid](patterns/waitid.md) — Async process waiting (IORING_OP_WAITID, 6.7) ✨⁶
- [Pipe Operation](patterns/pipe-op.md) — IORING_OP_PIPE async pipe creation (6.14) ✨⁷
- [Multishot URING_CMD](patterns/uring-cmd-multishot.md) — Generalized multishot for any driver (6.18) ✨⁷
- [Decision Guide](patterns/decision-guide.md) — When to use io_uring vs alternatives ✨⁷
- [Latency-Sensitive Workloads](patterns/realtime-latency.md) — Games, audio, trading: low-latency config ✨⁷
- [Performance Tuning](patterns/performance-tuning.md) — Ring sizing, batch strategies, CQ overflow, anti-patterns ✨⁸
- [Database Patterns](patterns/database-patterns.md) — WAL writes, page prefetch, fsync chains, group commit ✨⁸
- [Ring-to-Ring Messaging](patterns/ring-messaging.md) — MSG_RING, fd passing, cross-thread coordination ✨⁸
- [Splice Patterns](patterns/splice-patterns.md) — Zero-copy file↔socket↔socket via pipe ✨⁹
- [Timeout Patterns](patterns/timeout-patterns.md) — Absolute, multishot, link timeout, registered clock ✨⁹
- [Eventfd Integration](patterns/eventfd-integration.md) — Bridging io_uring to epoll/libuv/GLib event loops ✨⁹
- [Socket Lifecycle](patterns/socket-lifecycle.md) — Full async socket→bind→listen→accept→recv→send→close ✨⁹
- [Personality](patterns/personality.md) — Credential switching per operation ✨⁹
- [Cancel Patterns](patterns/cancel-patterns.md) — ASYNC_CANCEL flags, cancel by fd/opcode/any, sync cancel ✨¹⁰
- [Poll Operations](patterns/poll-ops.md) — POLL_ADD multishot, level/edge trigger, update ✨¹⁰
- [File Operations](patterns/file-ops.md) — Complete async file lifecycle: open, read, stat, xattr, fsync ✨¹⁰
- [Custom URING_CMD](patterns/uring-cmd-custom.md) — Writing your own URING_CMD driver handler ✨¹⁰
- [NAPI Busy Poll](patterns/napi-busy-poll.md) — Per-ring NAPI integration for sub-10µs network latency ✨¹¹
- [Buffer API Comparison](patterns/buffer-api-comparison.md) — provide_buffers vs pbuf_ring: old vs new ✨¹¹
- [EPOLL_CTL Operation](patterns/epoll-ctl-op.md) — Async epoll management from io_uring (5.6) ✨¹¹
- [Fixed Buffer Updates](patterns/fixed-buffer-updates.md) — Tagged registration, dynamic update, hot-swap, clone ✨¹¹
- [Signals](patterns/signals.md) — Signal masking, SIGCHLD replacement, signalfd integration ✨¹¹
- [AIO Migration](patterns/aio-migration.md) — Replacing libaio/linux-aio with io_uring ✨¹²
- [SQ/CQ Backpressure](patterns/backpressure.md) — Handling full rings, CQ overflow, batch strategies ✨¹²
- [TUN/TAP Devices](patterns/tuntap.md) — io_uring with TUN/TAP: what works, what doesn't ✨¹³
- [CQ Overflow Recovery](patterns/cq-overflow.md) — Overflow strategies, sizing, CQE_SKIP, monitoring ✨¹³
- [Thread-per-Core](patterns/thread-per-core.md) — Architecture patterns: ring config, resources, messaging ✨¹⁴
- [Async Filesystem Ops](patterns/async-fs-ops.md) — mkdir, rename, unlink, statx, xattr batch patterns ✨¹⁴

### [Benchmarks](benchmarks/)
- [io_uring vs epoll](benchmarks/iouring-vs-epoll.md) — The numbers that matter
- [io_uring vs aio](benchmarks/iouring-vs-aio.md) — Why aio is dead
- [io_uring vs DPDK](benchmarks/iouring-vs-dpdk.md) — Network I/O: kernel stack vs. bypass ✨✨
- [Real-World Data](benchmarks/real-world-data.md) — Published numbers from ScyllaDB, Netty, fio ✨⁶
- [Benchmarking Methodology](benchmarks/methodology.md) — fio configs, perf-stat, tracing, anti-patterns ✨¹⁴

### [Frameworks](frameworks/)
- [Overview](frameworks/overview.md) — Who's using io_uring and how
- [TigerBeetle](frameworks/tigerbeetle.md) — Financial database, Zig + io_uring
- [Glommio](frameworks/glommio.md) — Rust thread-per-core runtime
- [Monoio](frameworks/monoio.md) — Rust thread-per-core from ByteDance
- [tokio-uring](frameworks/tokio-uring.md) — io_uring backend for Tokio
- [io_uring-go](frameworks/io-uring-go.md) — Go bindings
- [Seastar](frameworks/seastar.md) — C++ thread-per-core, ScyllaDB/Redpanda ✨
- [FUSE over io_uring](frameworks/fuse-io-uring.md) — Kernel↔userspace filesystem communication ✨
- [Adoption Status](frameworks/adoption-status.md) — Who's using it, who isn't, and why ✨
- [TigerBeetle Source](frameworks/tigerbeetle-source.md) — Deep dive into io/linux.zig implementation ✨✨
- [Netty](frameworks/netty.md) — Java io_uring transport (Netty 4.2) ✨✨✨
- [PostgreSQL](frameworks/postgresql.md) — Async I/O subsystem with io_uring backend ✨✨✨
- [Cloudflare & Fastly](frameworks/cloudflare-fastly.md) — CDN production adoption patterns ✨⁴
- [Zig std.Io](frameworks/zig-std-io.md) — io_uring as first-class stdlib backend (Feb 2026) ✨⁵
- [Rust Ecosystem](frameworks/rust-ecosystem.md) — tokio-uring, monoio, glommio, compio, ringbahn survey ✨⁵
- [ScyllaDB](frameworks/scylladb.md) — Production io_uring at petabyte scale ✨⁶
- [ublk](frameworks/ublk.md) — Userspace block devices via io_uring passthrough ✨⁷
- [Compio](frameworks/compio.md) — Cross-platform completion-based Rust runtime (io_uring + IOCP) ✨⁸
- [Envoy](frameworks/envoy.md) — L7 proxy, experimental io_uring backend ✨⁹
- [QEMU](frameworks/qemu.md) — VM block I/O via io_uring (since QEMU 5.0) ✨⁹
- [HTTP Servers](frameworks/http-servers.md) — Adoption in Nginx, H2O, Drogon, Caddy, etc. ✨¹⁰
- [systemd & Proxies](frameworks/systemd-proxies.md) — Why systemd, HAProxy, and Traefik don't use io_uring ✨¹¹
- [Edge Compute](frameworks/edge-compute.md) — Cloudflare Workers, Deno Deploy, Wasm runtimes ✨¹¹
- [libuv / Node.js / Bun](frameworks/libuv-nodejs.md) — JS runtime io_uring adoption (file-only in libuv, none in Bun) ✨¹²
- [Distributed Storage](frameworks/distributed-storage.md) — Ceph, MinIO, RocksDB, FoundationDB ✨¹²
- [QEMU/KVM Virtio](frameworks/qemu-virtio.md) — io_uring in the virtualization stack, guest↔host path ✨¹²
- [Embedded Rust](frameworks/embedded-rust.md) — Embassy, probe-rs, and embedded Linux ✨¹³
- [Media/Video Processing](frameworks/media-video.md) — GStreamer, FFmpeg, DMA-BUF video pipelines ✨¹³
- [Rust Web Frameworks](frameworks/rust-web-frameworks.md) — Actix, Axum, Warp: why they all use epoll ✨¹⁴
- [Redpanda](frameworks/redpanda.md) — Seastar-based streaming, io_uring for log I/O ✨¹⁴

### [Pitfalls](pitfalls/)
- [Common Mistakes](pitfalls/common-mistakes.md) — What will bite you
- [Security](pitfalls/security.md) — Kernel attack surface and mitigations
- [BPF Filtering](pitfalls/bpf-filtering.md) — Fine-grained io_uring sandboxing ✨
- [Debugging](pitfalls/debugging.md) — Tracepoints, bpftrace, perf, ftrace recipes ✨✨
- [Cloud & Containers](pitfalls/cloud-containers.md) — Provider status, seccomp, enabling io_uring ✨✨
- [CVE Timeline](pitfalls/cve-timeline.md) — Security vulnerabilities, hardening timeline, production guidance ✨✨✨
- [Distro Support](pitfalls/distro-support.md) — RHEL disabled, Android locked, Docker blocked ✨⁶
- [Testing](pitfalls/testing.md) — liburing test suite, syzkaller fuzzing, CI patterns ✨⁶
- [Storage Infrastructure](pitfalls/storage-infra.md) — dm-crypt, LVM, bcachefs, XFS, NVMe passthrough ✨¹¹
- [WSL2](pitfalls/wsl2.md) — io_uring on Windows Subsystem for Linux: works, with caveats ✨¹³
- [Memory Overhead](pitfalls/memory-overhead.md) — Per-ring, per-request, registered resource costs ✨¹³
- [Container Runtimes](pitfalls/container-runtimes.md) — Docker, Kubernetes, seccomp: why io_uring is blocked ✨¹³
- [POSIX Compliance Gaps](pitfalls/posix-gaps.md) — io_uring vs POSIX: 12 behavior divergences ✨¹⁴
- [Android](pitfalls/android.md) — GKI kernel, SELinux, seccomp: why io_uring is locked out ✨¹⁴

### [Resources](resources/)
- [Links](resources/links.md) — Papers, talks, articles, source code
- [Glossary](resources/glossary.md) — Quick reference for io_uring terminology ✨⁹
- [Cheat Sheet](resources/cheatsheet.md) — Quick reference card: setup, ops, flags, kernel versions ✨¹²

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
| Waitid | 6.7 |
| Futex ops | 6.7 |
| NAPI busy poll | 6.9 |
| Incremental buffer consumption (`IOU_PBUF_RING_INC`) | 6.10 |
| FIXED_FD_INSTALL | 6.10 |
| Hybrid IOPOLL | 6.13 |
| Ring resize | 6.13 |
| Clone buffers | 6.13 |
| Registered wait regions | 6.13 |
| Bind/Listen ops | 6.14 |
| FUSE over io_uring | 6.14 |
| R/W integrity/PI metadata | 6.14 |
| Pipe op | 6.14 |
| Zero-copy RX (zcrx) | 6.15 |
| epoll_wait via io_uring | 6.15 |
| Vectored registered buffers | 6.15 |
| NO_SQARRAY | 6.7 |
| Bundle recv/send | 6.10 |
| Socket URING_CMD (getsockopt, etc.) | 6.11 |
| TX timestamps | 6.13 |
| Mixed CQE sizes (`CQE_MIXED`) | 6.18 |
| Mixed SQE sizes (`SQE_MIXED`) | 6.19 |
| SQ Rewind | 6.19 |
| Multiple zcrx IFQs | 6.16 |
| DMA-BUF zcrx | 6.16 |
| BPF filtering | 6.19 |
| REGISTER_QUERY | 6.19 |
| zcrx queries / ctrl | 6.19 |
| getsockname/getpeername via URING_CMD | 6.19 |
| REGISTER_ZCRX_CTRL | 6.19 |
| REGISTER_SEND_MSG_RING | 6.10 |
| REGISTER_CLONE_BUFFERS | 6.10 |
| REGISTER_CLOCK | 6.10 |
| Multishot URING_CMD | 6.18 |
| ublk batch I/O | 6.18 |
| ublk zero-copy | 6.15 |
| REGISTER_PBUF_STATUS | 6.4 |
| MSG_RING | 5.18 |
| MSG_RING fd passing | 6.0 |
| ASYNC_CANCEL by fd | 5.19 |
| ASYNC_CANCEL_ANY | 6.0 |
| ASYNC_CANCEL by opcode | 6.1 |
| SYNC_CANCEL | 6.0 |
| POLL_ADD multishot | 5.13 |
| POLL_ADD level-triggered | 5.18 |
| xattr ops (get/set) | 5.19 |
| ftruncate | 6.9 |
| REGISTER_NAPI (busy poll) | 6.9 |
| EPOLL_CTL | 5.6 |
| Vectored registered R/W (READV_FIXED/WRITEV_FIXED) | 6.15 |

## Contributing

Open issues. Don't submit PRs with tutorial-level content.
