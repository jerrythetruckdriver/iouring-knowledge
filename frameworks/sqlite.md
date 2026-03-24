# SQLite and io_uring

## Status: No Native Support

SQLite has no io_uring VFS as of March 2026. The deprecated `sqlite3async.c` extension (thread-based async writes) was abandoned in favor of WAL mode.

## Why It Doesn't Exist

1. **WAL mode solved the main problem.** WAL + `PRAGMA synchronous=NORMAL` avoids fsync on every commit. Only checkpoints fsync. The async I/O module was deprecated because WAL made it unnecessary.

2. **Single-writer model.** SQLite is fundamentally single-writer. io_uring's strength — batching concurrent I/O — doesn't help when there's one writer.

3. **Cross-platform requirement.** SQLite runs everywhere: Windows, macOS, embedded, WASM. A Linux-only VFS doesn't fit the project's priorities.

4. **VFS abstraction.** SQLite's VFS layer (xRead, xWrite, xSync, xLock) is synchronous by design. An io_uring VFS would need to block in xWrite/xRead until the CQE arrives — negating the async advantage unless the VFS API itself changes.

## Where io_uring Could Help SQLite

### WAL Checkpoint

Checkpoints write dirty pages from WAL back to the database. Multiple pages can be written concurrently:

```
// Theoretical: batch checkpoint writes
for each dirty page:
    io_uring_prep_write(sqe, db_fd, page, PAGE_SIZE, offset);
io_uring_submit();
// Wait for all completions
io_uring_prep_fsync(sqe, db_fd, 0);
```

This would parallelize random writes during checkpoint. Real benefit on NVMe.

### Batch Read for Large Scans

Table scans or index traversals could prefetch multiple pages:

```
for each needed page:
    io_uring_prep_read(sqe, db_fd, buf, PAGE_SIZE, offset);
io_uring_submit();
```

### WAL Write + Fsync Chain

```
WRITE(wal_frame) → [link] → FSYNC(wal_fd)
```

Linked write+fsync eliminates the syscall gap between write and fsync.

## Third-Party Efforts

A few experimental io_uring SQLite VFS implementations exist on GitHub. None are production-quality:

- **Approach 1: Synchronous wrapper.** Submit SQE, immediately wait for CQE. Gains registered buffers/files, loses async benefit. Marginal improvement.
- **Approach 2: Batched VFS.** Buffer writes, submit in batches on xSync. Better throughput but complex correctness (what if xRead needs data from buffered write?).
- **Approach 3: Rewrite around completion.** Abandon SQLite's synchronous VFS model entirely. At that point you're not really using SQLite anymore.

## Comparison With Other Databases

| Database | io_uring | Why |
|----------|----------|-----|
| SQLite | No | Single-writer, cross-platform, VFS is sync |
| PostgreSQL | In progress | Multi-process, heavy WAL, huge benefit |
| ScyllaDB | Yes | Thread-per-core, built for async |
| ClickHouse | Yes | Column store, heavy sequential reads |
| TigerBeetle | Yes | Built from scratch around io_uring |

## The Honest Take

SQLite doesn't need io_uring for most workloads. WAL mode + mmap + O_DIRECT (where supported) covers the performance-sensitive cases. The VFS abstraction would need a fundamental redesign to benefit from completion-based I/O, and SQLite's priorities are correctness and portability, not maximum throughput.

If your SQLite workload is I/O-bound enough that you need io_uring, you probably need a different database.
