# Database I/O Patterns

How databases use (or should use) io_uring. WAL writes, page prefetch, fsync chains, and the patterns that actually matter for durability and throughput.

## WAL Write Path

Write-Ahead Logging is the critical durability path. Every committed transaction hits the WAL before anything else.

### Traditional (synchronous)
```
write(wal_fd, data, len);
fdatasync(wal_fd);
// block until durable — commits stall here
```

### io_uring (linked write + fsync)
```c
// SQE 1: write
sqe = io_uring_get_sqe(ring);
io_uring_prep_write(sqe, wal_fd, data, len, offset);
sqe->flags |= IOSQE_IO_LINK;  // chain to next
sqe->user_data = WAL_WRITE;

// SQE 2: fsync (runs after write completes)
sqe = io_uring_get_sqe(ring);
io_uring_prep_fsync(sqe, wal_fd, IORING_FSYNC_DATASYNC);
sqe->user_data = WAL_FSYNC;

io_uring_submit(ring);
```

The link guarantees ordering. One submission, no blocking. The commit is durable when the fsync CQE arrives.

### Group Commit with Batching

Multiple transactions can share a WAL fsync:

```c
// Accumulate WAL writes from multiple transactions
for (tx : pending_transactions) {
    sqe = io_uring_get_sqe(ring);
    io_uring_prep_write(sqe, wal_fd, tx->data, tx->len, wal_offset);
    sqe->flags |= IOSQE_IO_LINK;
    wal_offset += tx->len;
}

// Single fsync covers all writes (hard-linked to last write)
sqe = io_uring_get_sqe(ring);
io_uring_prep_fsync(sqe, wal_fd, IORING_FSYNC_DATASYNC);

io_uring_submit(ring);
// All N transactions become durable with one fsync
```

This is the PostgreSQL pattern — Andres Freund's async I/O work uses exactly this approach.

## Page Prefetch

B-tree traversals and sequential scans benefit from read-ahead. io_uring's async read makes prefetch trivial:

```c
// Prefetch pages we'll need soon
for (int i = 0; i < prefetch_count; i++) {
    sqe = io_uring_get_sqe(ring);
    io_uring_prep_read(sqe, data_fd, page_buf[i], PAGE_SIZE, page_offset[i]);
    sqe->flags |= IOSQE_FIXED_FILE;  // registered fd
    sqe->buf_index = buf_idx[i];      // registered buffer
    sqe->user_data = PREFETCH | page_id[i];
}
io_uring_submit(ring);

// Continue processing current pages while prefetch happens in background
// Check CQEs when we actually need the prefetched pages
```

### With Registered Buffers

For hot-path page reads, register a buffer pool:

```c
struct iovec buffers[BUFFER_POOL_SIZE];
for (int i = 0; i < BUFFER_POOL_SIZE; i++) {
    buffers[i].iov_base = aligned_alloc(4096, PAGE_SIZE);
    buffers[i].iov_len = PAGE_SIZE;
}
io_uring_register_buffers(ring, buffers, BUFFER_POOL_SIZE);

// Now use io_uring_prep_read_fixed() — skip buffer import overhead
```

Saves ~100-200ns per read. At 100K page reads/sec, that's 10-20ms of CPU per second.

## Checkpoint / Flush

Dirty page writeback for checkpointing:

```c
// Write dirty pages — can be heavily parallelized
for (page : dirty_pages) {
    sqe = io_uring_get_sqe(ring);
    io_uring_prep_write_fixed(sqe, data_fd, page->buf_idx, PAGE_SIZE, page->offset, page->buf_idx);
    sqe->user_data = CHECKPOINT | page->id;
}
io_uring_submit(ring);

// Wait for all writes to complete
while (outstanding > 0) {
    io_uring_wait_cqe(ring, &cqe);
    // ... process completions, decrement outstanding
}

// Final fsync ensures durability
sqe = io_uring_get_sqe(ring);
io_uring_prep_fsync(sqe, data_fd, IORING_FSYNC_DATASYNC);
io_uring_submit(ring);
io_uring_wait_cqe(ring, &cqe);
```

### With IOPOLL

For NVMe with O_DIRECT:

```c
// Setup with IOPOLL for checkpoint writes
IORING_SETUP_IOPOLL | IORING_SETUP_SQPOLL
```

Polled completions avoid interrupt overhead. ScyllaDB uses this for compaction and flush — see `frameworks/scylladb.md`.

## fsync Chains

Pattern: multiple files need durable writes (WAL + metadata + data files).

```c
// Write WAL
sqe = prep_write(wal_fd, wal_data); sqe->flags |= IOSQE_IO_LINK;
sqe = prep_fsync(wal_fd);           sqe->flags |= IOSQE_IO_LINK;

// Write metadata (depends on WAL durability)
sqe = prep_write(meta_fd, meta_data); sqe->flags |= IOSQE_IO_LINK;
sqe = prep_fsync(meta_fd);

io_uring_submit(ring);
```

The link chain enforces: WAL write → WAL fsync → meta write → meta fsync. Four operations, one submission, correct ordering.

**Caution**: Soft links (`IOSQE_IO_LINK`) cancel the chain on failure. If the WAL write fails, the metadata write won't happen. Usually correct for databases. Use hard links (`IOSQE_IO_HARDLINK`) if you need the chain to continue regardless.

## Index Maintenance

B-tree splits, LSM compaction — these are write-heavy. io_uring patterns:

### Parallel SSTable Writes (LSM)
```c
// Write multiple SSTables concurrently during compaction
for (sst : output_sstables) {
    for (block : sst->blocks) {
        sqe = io_uring_get_sqe(ring);
        io_uring_prep_write(sqe, sst->fd, block->data, block->len, block->offset);
        sqe->user_data = encode(sst->id, block->id);
    }
}
io_uring_submit(ring);
// Process CQEs as blocks complete — track per-SSTable progress
```

### NVMe Passthrough

For databases that want maximum control:

```c
// Direct NVMe write command via URING_CMD
sqe = io_uring_get_sqe(ring);
io_uring_prep_cmd(sqe, NVME_URING_CMD_IO, nvme_fd, ...);
```

Bypasses the block layer entirely. TigerBeetle explored this for its storage engine.

## Who's Doing What

| Database | io_uring Usage | Status |
|----------|---------------|--------|
| ScyllaDB | Full: IOPOLL, registered buffers, NVMe passthrough | Production |
| TigerBeetle | Core I/O engine, custom ring management | Production |
| PostgreSQL | Pluggable async I/O backend (WAL, prefetch) | In development |
| RocksDB | Under discussion, no implementation yet | Planning |
| FoundationDB | Not using io_uring | — |
| CockroachDB | Go runtime limitations | — |
| SQLite | WAL mode could benefit; no plans | — |

## Key Takeaways

1. **Linked write+fsync** is the fundamental database pattern — one submission for durable writes
2. **Group commit** via batched writes + single fsync is free throughput
3. **Prefetch with registered buffers** eliminates per-read overhead for page cache misses
4. **IOPOLL + O_DIRECT** for storage-bound workloads on NVMe
5. Go and JVM databases face runtime impedance — io_uring benefits mostly C/C++/Rust/Zig
