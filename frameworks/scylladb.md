# ScyllaDB — io_uring in Production at Scale

C++ NoSQL database. Thread-per-core (Seastar framework). One of the earliest and most sophisticated io_uring adopters.

## Architecture

ScyllaDB runs on the [Seastar framework](seastar.md). Each shard (CPU core) has its own io_uring ring. No sharing, no locking.

```
[Shard 0] ←→ [io_uring ring 0] ←→ kernel
[Shard 1] ←→ [io_uring ring 1] ←→ kernel
[Shard N] ←→ [io_uring ring N] ←→ kernel
```

## Migration Path: linux-aio → io_uring

ScyllaDB was already using linux-aio before io_uring existed. The migration preserved the async-by-design architecture but unlocked:

1. **Beyond O_DIRECT** — linux-aio only works with O_DIRECT. io_uring works with everything.
2. **Batched syscalls** — submit multiple heterogeneous operations in one `io_uring_enter()`
3. **File/buffer registration** — eliminate per-operation fd lookup and buffer mapping overhead
4. **Poll ring** — for NVMe devices, avoid interrupt overhead entirely

## Performance Numbers (from ScyllaDB benchmarks)

Benchmark: 1kB random reads, 8 CPUs, 72 fio jobs, iodepth 8, NVMe storage:

| Backend | IOPS | Context Switches | vs io_uring |
|---------|------|------------------|-------------|
| sync | 814K | 27.6M | -42.6% |
| posix-aio (threads) | 433K | 64.1M | -69.4% |
| linux-aio | 1,322K | 10.1M | -6.7% |
| io_uring (basic) | 1,417K | 11.3M | baseline |
| io_uring (enhanced) | 1,486K | 11.5M | +4.9% |

"Enhanced" = file registration + buffer registration + poll ring.

Key takeaway: For applications already using linux-aio, io_uring gains are modest (~7%). The real win is for everyone else — **69% improvement** over thread pools.

## Advanced Features Used

### IOPOLL Mode
For NVMe devices with ultra-low latency (single-digit μs), interrupt processing itself becomes the bottleneck. ScyllaDB uses `IORING_SETUP_IOPOLL` to poll for completions instead.

### Registered Buffers + Files
Pre-register DMA buffers and file descriptors to eliminate per-I/O kernel mapping overhead. Matters at millions of IOPS.

### NVMe Passthrough (URING_CMD)
For workloads that want to bypass the block layer entirely. Direct NVMe command submission via `IORING_OP_URING_CMD`. ScyllaDB has experimented with this for specific storage paths.

## The ScyllaDB + eBPF Vision

Glauber Costa (ScyllaDB) articulated the convergence thesis in 2020:

> eBPF pushes code *into* the kernel. io_uring pulls kernel work *out* asynchronously. Together, they make the kernel a programmable async coprocessor.

See [internals/ebpf-integration.md](../internals/ebpf-integration.md).

## Why ScyllaDB Matters for io_uring

- **Production validation** at petabyte scale
- **Seastar framework** is the reference thread-per-core io_uring integration
- **Benchmark data** that's real, not synthetic
- **Drove kernel improvements** — ScyllaDB engineers have contributed io_uring patches

## Sources

- Glauber Costa, "How io_uring and eBPF Will Revolutionize Programming in Linux" (May 2020)
- ScyllaDB blog, various io_uring performance posts
- Seastar framework io_uring backend source
