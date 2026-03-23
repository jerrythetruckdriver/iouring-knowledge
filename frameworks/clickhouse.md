# ClickHouse — io_uring for Async Reads

ClickHouse has a dedicated `IOUringReader` class (`src/Disks/IO/IOUringReader.cpp`). It's one of the cleaner io_uring integrations in the database world.

## Architecture

Single ring per process (singleton). One monitor thread polls CQEs. Reads only — no writes, no fsync.

```
Application threads → submit(Request) → promise/future
                                           ↓
                              IOUringReader (mutex-protected)
                                           ↓
                              io_uring ring (IORING_OP_READ)
                                           ↓
                              monitorRing() thread → CQE → fulfill promise
```

## What It Uses

| Feature | Status |
|---------|--------|
| IORING_OP_READ | ✅ Only operation |
| Probe API | ✅ Checks IORING_OP_READ support |
| Short read resubmission | ✅ Tracks bytes_read, resubmits |
| EAGAIN/EINTR retry | ✅ Both submit and completion sides |
| Registered buffers | ❌ |
| Registered files | ❌ |
| SQPOLL | ❌ |
| Linked ops | ❌ |
| Multishot | ❌ |

## Key Design Decisions

**CQ-sized in-flight cap.** Tracks `in_flight_requests.size() < cq_entries`. Excess goes to a `pending_requests` deque. On completion, drains pending queue FIFO. Prevents CQ overflow entirely.

**Buffer address as request ID.** Uses `reinterpret_cast<UInt64>(request.buf)` as user_data. Assumes no two concurrent reads target the same buffer. Simple but effective.

**Mutex around everything.** Both submit and monitor threads lock. Not great for throughput, but ClickHouse's bottleneck is decompression/query processing, not I/O submission.

**NOP for shutdown.** Sends a NOP with null user_data to wake the monitor thread on destruction. Clean teardown pattern.

**Memory tracking.** Calls `io_uring_mlock_size_params()` to get exact ring memory cost, attributes it to global memory tracker. Good operational practice.

## What's Missing

No registered files — every read pays the fd lookup cost. No registered buffers — every read pins pages. No SQPOLL — explicit submit every time. No linked write+fsync chains for MergeTree mutations.

For a columnar store that does heavy sequential reads from large files (`.bin` column files), registered files + provided buffer rings would be the obvious next step. But ClickHouse's query engine is CPU-bound on decompression and aggregation — the I/O layer is rarely the bottleneck.

## Configuration

```xml
<!-- ClickHouse config -->
<use_io_uring_for_disk_read>true</use_io_uring_for_disk_read>
```

Disabled by default. Compile-time `USE_LIBURING` gate.

## Verdict

Conservative but correct. ClickHouse uses io_uring as a drop-in async read backend, not as an architecture driver. The short read handling and CQ overflow prevention are well-implemented. The mutex serialization is the main scaling limit, but for ClickHouse's workload (few large reads, CPU-bound processing) it's fine.

Compare with ScyllaDB (one ring per shard, IOPOLL, registered buffers) or TigerBeetle (custom three-queue architecture) — ClickHouse is the "minimum viable io_uring" approach. And that's actually a valid engineering choice.

## Source

- `ClickHouse/src/Disks/IO/IOUringReader.h` — header, 90 lines
- `ClickHouse/src/Disks/IO/IOUringReader.cpp` — implementation, 390 lines
