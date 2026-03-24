# mmap vs io_uring for Random Reads

## The Question

For random read workloads (databases, search indexes, key-value stores), mmap and io_uring represent fundamentally different I/O models. This page compares them honestly.

## How They Work

### mmap

```
addr = mmap(NULL, size, PROT_READ, MAP_SHARED, fd, 0);
data = addr[offset];  // page fault if not cached
```

On cache hit: direct memory access, zero syscalls. On cache miss: page fault → kernel → block I/O → returns.

### io_uring

```
sqe->opcode = IORING_OP_READ;
sqe->fd = fd;
sqe->off = offset;
sqe->addr = buffer;
sqe->len = len;
```

Always involves SQE submission and CQE completion. But: batches multiple reads, handles cache misses explicitly, supports O_DIRECT.

## When mmap Wins

**Working set fits in page cache:**
- Cache hit = raw memory load (nanoseconds)
- io_uring can't compete with direct memory access
- No syscall, no SQE/CQE overhead

**Sequential/predictable access with kernel readahead:**
- Kernel's readahead detects sequential patterns automatically
- Prefaults pages ahead of your access pattern
- mmap + `madvise(MADV_SEQUENTIAL)` is hard to beat

## When io_uring Wins

**Working set exceeds memory (random access):**
- mmap page faults are synchronous and uncontrollable
- One thread = one outstanding I/O at a time (blocked on fault)
- io_uring batches dozens of reads in a single submission
- io_uring + O_DIRECT bypasses page cache entirely

**The page fault problem:**
- Page faults are *invisible* latency — no way to control or predict them
- Major faults block the thread until I/O completes
- Under memory pressure, the kernel evicts pages and you re-fault
- No way to batch, prioritize, or cancel faults

**Numbers (typical NVMe, random 4KB reads, cold cache):**

| Method | Concurrent I/Os | IOPS (approx) |
|--------|----------------|---------------|
| mmap (1 thread) | 1 (blocked on fault) | ~50K |
| mmap (32 threads) | 32 (thread per fault) | ~500K |
| io_uring (1 thread, QD=128) | 128 | ~800K+ |
| io_uring + IOPOLL (1 thread) | 128 | ~1M+ |

The fundamental issue: mmap achieves concurrency through threads. io_uring achieves it through queue depth on a single thread.

## The Database Consensus

Most modern databases that have evaluated both choose io_uring (or direct I/O) for their storage engine:

- **ScyllaDB:** io_uring + O_DIRECT. Explicit I/O control.
- **TigerBeetle:** io_uring + O_DIRECT. Deterministic latency.
- **ClickHouse:** io_uring for async reads (IOUringReader).
- **PostgreSQL:** Historically mmap (for shared buffers), moving toward io_uring for prefetch.
- **RocksDB:** Still mmap for random reads, but increasingly recommends pread.
- **LMDB:** mmap-only (by design). Works well when dataset fits in RAM.
- **LevelDB/Pebble (Go):** pread. Can't use io_uring (Go runtime).

## Hybrid Approach

Some systems use both:
1. mmap for hot/cached data (zero-copy reads)
2. io_uring for prefetch/cold data (explicit async I/O)
3. `madvise(MADV_WILLNEED)` as a middle ground (async-ish, but limited)

## Summary

| Factor | mmap | io_uring |
|--------|------|----------|
| Cache hit latency | ~ns (memory load) | ~μs (SQE→CQE) |
| Cache miss control | None (page fault) | Full (explicit I/O) |
| Concurrency model | Threads | Queue depth |
| O_DIRECT | No | Yes |
| Batching | No | Yes |
| Memory overhead | Page tables + cache | User buffers |
| Portability | Universal | Linux 5.1+ |
| Complexity | Simple | Moderate |

**Rule of thumb:** If your working set fits in RAM, mmap is fine. If it doesn't, io_uring + O_DIRECT gives you control that mmap fundamentally can't.
