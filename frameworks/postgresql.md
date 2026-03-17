# PostgreSQL and io_uring

## Status: Active Development (2024-2026)

Andres Freund (Microsoft/PostgreSQL) has been driving an asynchronous I/O subsystem for PostgreSQL that includes io_uring as a backend. This is one of the most significant ongoing PostgreSQL infrastructure efforts.

## The Problem

PostgreSQL's I/O model is synchronous. Every `pread()`/`pwrite()` blocks the backend process. For WAL writes, this means:

1. Format WAL record in shared memory
2. `write()` to WAL segment — blocks
3. `fdatasync()` — blocks again
4. Only then acknowledge to client

Under heavy write loads, backends spend significant time waiting on I/O. The buffer pool (`shared_buffers`) also uses synchronous reads — a cache miss stalls the query.

## The AIO Subsystem

PostgreSQL is building an internal async I/O layer (`src/backend/storage/aio/`) with pluggable backends:

| Backend | Description |
|---------|-------------|
| `posix_aio` | POSIX AIO (portable, limited) |
| `io_uring` | Linux io_uring |
| `worker` | Thread pool doing sync I/O (fallback) |

### io_uring Backend

The PostgreSQL io_uring backend:
- One ring per backend process (not per database)
- Uses `IORING_SETUP_SQPOLL` for WAL writes to avoid syscall overhead
- Registered buffers for buffer pool pages (skip kernel copy)
- Batches related operations (e.g., prefetch next pages while processing current)

### WAL Write Path (Proposed)

```
Old: write() → fdatasync() → return     [2 syscalls, both blocking]
New: prep_write(sqe) → prep_fsync(sqe)  [linked ops, 1 submit]
     → process other work while I/O is in flight
     → harvest CQE when needed
```

The linked write+fsync pattern lets PostgreSQL overlap WAL writes with query processing. At commit time, just check if the CQE arrived rather than issuing a blocking sync.

### Buffer Pool Prefetch

More impactful than WAL: async prefetch of buffer pool pages. When the executor knows it'll need pages soon (sequential scan, index scan), it can submit io_uring reads ahead of time:

```
Scan page N → prep_read(page N+8) → process page N → page N+8 already in cache
```

This is where io_uring's batching really shines. Submit 16 prefetch reads in one `io_uring_enter()`, zero additional syscalls.

## Timeline

- **2020:** Initial async I/O RFC by Andres Freund
- **2022-2023:** Substantial rework of the buffer manager to support async paths
- **2024:** AIO infrastructure patches posted to pgsql-hackers, multiple commitfest entries
- **2025-2026:** Iterating toward merge. The core AIO framework is solid; the integration points (bgwriter, checkpointer, WAL writer) are being refined.

**Not merged yet.** This is a multi-year effort touching some of PostgreSQL's oldest code. The io_uring backend exists and works, but PostgreSQL's conservative merge policy means extensive review.

## Impact Potential

PostgreSQL on io_uring could be transformative for write-heavy workloads:
- WAL write latency reduction: 30-50% estimated (fewer syscalls, async fsync)
- Sequential scan speedup: significant on cold cache (prefetch eliminates stalls)
- Checkpoint performance: async writeback of dirty pages

The biggest win isn't raw I/O speed — it's **latency hiding**. PostgreSQL can do useful work while I/O is in flight instead of blocking.

## What About RocksDB?

RocksDB (used by CockroachDB, TiKV, etc.) has experimented with io_uring for compaction I/O but hasn't shipped it. Their `PosixRandomAccessFile` and `PosixWritableFile` are deeply synchronous. The WAL in RocksDB (`Writer::WriteToWAL`) would benefit most, but the refactoring cost is high for marginal gains — RocksDB already does well with direct I/O and io_submit (legacy AIO).

## Why This Matters

PostgreSQL is the most popular open-source database. When it ships io_uring support, it validates io_uring for the single largest category of Linux I/O workloads: databases. Every other database team will take notice.
