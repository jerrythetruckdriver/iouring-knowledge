# Seastar Networking Stack — io_uring Deep Dive

## Architecture

Seastar is the C++ framework behind ScyllaDB and Redpanda. Thread-per-core with one io_uring ring per shard (thread). No locks, no shared state between shards.

### Ring Configuration

Each shard creates a ring with:

```
IORING_SETUP_COOP_TASKRUN | IORING_SETUP_SINGLE_ISSUER
```

Single issuer because exactly one thread owns each ring. Cooperative taskrun because Seastar controls its own event loop timing.

### Event Loop Integration

Seastar's reactor (event loop) integrates io_uring alongside:

- epoll (for legacy compatibility, signals, timerfd)
- aio (Linux AIO, being phased out)
- io_uring (preferred path for storage)

The reactor polls all three in its main loop. io_uring CQEs are harvested non-blocking with `io_uring_peek_cqe()`.

## Storage Path (Primary io_uring Use)

Seastar uses io_uring primarily for storage, not networking. The storage path:

1. **O_DIRECT everywhere.** No page cache. Seastar manages its own buffer cache.
2. **Registered buffers.** DMA-aligned buffers registered once, used across all I/O.
3. **IOPOLL.** Polled I/O completion for NVMe devices. No interrupt overhead.
4. **Read/write batching.** Queue depth managed per-shard with I/O scheduling.

### I/O Scheduler

Seastar has a per-shard I/O scheduler that:
- Rate-limits submissions (disk bandwidth, IOPS caps)
- Prioritizes I/O classes (compaction vs client reads)
- Tracks latency percentiles per device
- Adjusts queue depth dynamically

This sits *above* io_uring. The scheduler decides what to submit; io_uring executes it.

## Networking Path (Mostly Not io_uring)

Seastar's networking historically uses its own userspace TCP stack (virtio-based) or the kernel's TCP via epoll. Networking is **not** the primary io_uring use case.

### Why Not io_uring for Networking?

1. **Seastar's own stack.** The userspace TCP stack (for DPDK/virtio) is faster than any kernel path. io_uring can't compete with zero-kernel networking.
2. **epoll is fine.** For the kernel TCP path (POSIX stack), epoll's overhead is negligible compared to TCP processing. Switching to io_uring saves ~200ns per event — invisible at application level.
3. **Mature codebase.** The epoll networking path has years of production hardening. Rewriting for marginal gain isn't worth the risk.

### Where io_uring Networking Could Help

- **Connection-heavy workloads**: Multishot accept + recv would reduce syscalls for thousands of idle connections. ScyllaDB typically has few long-lived connections, so this doesn't matter much.
- **Zero-copy receive (zcrx)**: For high-bandwidth ingest (>10Gbps), zcrx could eliminate copies. Not adopted yet.

## NVMe Passthrough

ScyllaDB uses `IORING_OP_URING_CMD` for NVMe passthrough when available:

- Bypasses block layer entirely
- Direct NVMe command submission
- Combined with IOPOLL for poll-mode completion
- Maximum IOPS on raw NVMe devices

This is the most advanced io_uring feature ScyllaDB uses.

## What Seastar Uses

| Feature | Used | Context |
|---------|------|---------|
| Basic read/write | ✅ | Storage, O_DIRECT |
| IOPOLL | ✅ | NVMe polled I/O |
| Registered buffers | ✅ | DMA-aligned I/O |
| Registered files | ✅ | Fixed file descriptors |
| COOP_TASKRUN | ✅ | Single-threaded reactor |
| SINGLE_ISSUER | ✅ | One thread per ring |
| NVMe passthrough | ✅ | URING_CMD on /dev/ng* |
| Multishot accept | ❌ | Low connection count |
| Multishot recv | ❌ | epoll networking path |
| Zero-copy send/recv | ❌ | Userspace stack preferred |
| Provided buffers | ❌ | Own buffer management |
| SQPOLL | ❌ | Reactor does its own polling |
| Linked ops | Minimal | Mostly independent submissions |

## What ScyllaDB and Redpanda Get From io_uring

The measurable wins:

1. **Batched submission**: Multiple reads/writes per io_uring_enter() call. Was the main motivator for migrating from Linux AIO.
2. **IOPOLL latency**: Sub-microsecond completion on NVMe without interrupt context switches.
3. **NVMe passthrough**: ~15-30% more IOPS than going through the block layer on p99 tails.
4. **Unified I/O path**: Storage + some internal ops through one interface.

## Comparison: Seastar vs Pure io_uring Server

Seastar uses io_uring as a storage engine. A pure io_uring server (like something built on Glommio or raw liburing) would use io_uring for *everything* — networking, timers, signals, IPC. Different philosophies:

- **Seastar**: io_uring is one I/O backend among several. Application logic drives the architecture.
- **Pure io_uring**: The ring *is* the event loop. Everything flows through SQ/CQ.

Neither is wrong. Seastar predates io_uring by years and has battle-tested alternatives for networking. New projects built from scratch would likely go all-in on io_uring.
