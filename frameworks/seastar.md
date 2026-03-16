# Seastar

C++ framework. Thread-per-core, shared-nothing architecture. Used by ScyllaDB and Redpanda.

**Repository**: [scylladb/seastar](https://github.com/scylladb/seastar)

## io_uring Integration

Seastar added io_uring as a backend alongside its existing AIO and epoll paths. The io_uring reactor is the preferred backend on supported kernels.

### What it uses

- **File I/O**: Replaced Linux AIO with io_uring for disk operations. AIO had the 64KB alignment restriction and didn't support buffered I/O. io_uring handles both direct and buffered.
- **Networking**: Still primarily uses epoll for networking via its own reactor. io_uring networking is available but not default in all configurations.
- **SQE batching**: Seastar's reactor naturally accumulates work, then flushes as a batch — fits io_uring's submission model perfectly.
- **SQPOLL**: Optional. ScyllaDB workloads are already CPU-pinned, so SQPOLL's benefit is marginal vs. the dedicated CPU cost.

### Architecture Notes

Seastar's shared-nothing model maps cleanly to io_uring:
- One ring per shard (CPU core)
- No cross-ring communication needed for I/O
- Each shard owns its registered files and buffers

### Why it matters

Seastar proved that io_uring's batching model can replace AIO entirely for database storage engines. ScyllaDB saw measurable latency improvements after the switch — fewer syscalls per I/O batch, and no more AIO's artificial 64KB request size ceiling.

## Key Design Decisions

1. **Dual backend**: Seastar maintains both AIO and io_uring paths. io_uring is preferred but AIO is fallback for older kernels.
2. **Conservative feature adoption**: Seastar doesn't rush to adopt every new io_uring feature. They validate on ScyllaDB workloads first.
3. **No multishot**: As of writing, Seastar doesn't use multishot accept/recv. Their reactor model resubmits per-event anyway.

## Performance Impact

ScyllaDB (Seastar-based) with io_uring vs AIO:
- Reduced syscall overhead on mixed read/write workloads
- Better tail latency due to batched submissions
- Eliminated AIO's alignment restrictions for certain operations
