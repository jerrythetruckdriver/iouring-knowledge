# LevelDB and Pebble (Go LSM Implementations)

## TL;DR

Neither LevelDB nor Pebble uses io_uring. LevelDB is C++ but minimalist. Pebble is Go, which means io_uring is off the table.

## LevelDB

Google's original LSM-tree key-value store. Pure C++, ~15K lines.

**I/O model:** Synchronous `pread()`/`pwrite()` with a background compaction thread. No event loop, no async I/O. The Env abstraction allows custom I/O backends, but nobody has shipped an io_uring Env.

**Why not io_uring:**
- LevelDB is essentially finished (maintenance-only since ~2017)
- I/O pattern is sequential writes (WAL, SST files) + random reads (point lookups)
- Sequential writes don't benefit much from io_uring (already fast with buffered I/O)
- Random reads could benefit, but LevelDB relies on mmap for table reads (`table/table.cc`)

## Pebble

CockroachDB's Go LSM engine, inspired by RocksDB/LevelDB. Production-critical.

**I/O model:** Standard Go `os.File` operations. `preadv2` on Linux when available. Background goroutines for compaction.

**Why not io_uring:** Go runtime limitation. Same reason as every Go database — goroutines + epoll netpoller can't integrate with io_uring's completion model without cgo overhead. See [io-uring-go.md](io-uring-go.md).

Pebble's approach to I/O performance:
- Direct I/O (`O_DIRECT`) for sstable writes and WAL
- `fdatasync()` for durability
- Read-ahead hints for iterators
- Block cache in Go heap
- Concurrent compactions via goroutines

The I/O bottleneck in Pebble isn't syscall overhead — it's compaction throughput and Go's garbage collector pausing during memory-intensive operations.

## RocksDB (for comparison)

C++, Facebook. Could theoretically use io_uring.

**Current state:** `MultiRead()` uses `preadv2()` with thread pool. No io_uring backend shipped. There was a [prototype patch](https://github.com/facebook/rocksdb/pull/7672) in 2020 that added io_uring for `MultiGet()`, achieving ~2x improvement in multi-read scenarios. It was merged but later reverted due to complexity.

RocksDB's `Env` abstraction supports pluggable I/O, and the `FileSystem` interface could accommodate io_uring, but maintainer interest has been low. The argument: for SSD workloads, the difference between preadv2 and io_uring for scattered reads is measurable but not transformative.

## The LSM Pattern

LSM-tree I/O is dominated by:

1. **WAL writes** — Sequential, fsync'd. io_uring's linked write+fsync helps here.
2. **SST file writes** — Sequential, large. Marginal io_uring benefit.
3. **Point lookups** — Random reads across multiple SST files. This is where io_uring shines — batch multiple pread calls.
4. **Compaction** — Sequential read + merge + sequential write. CPU-bound (key comparison, compression). I/O is already efficient with large reads.
5. **Iterator scans** — Sequential within a file. Kernel readahead handles this.

io_uring's biggest win for LSM engines would be **multi-key lookups** (reading from multiple SST files simultaneously) and **WAL write+fsync chains**. But most LSM engines amortize these through batching already.

## Who Does Use io_uring for LSM?

- **ScyllaDB** — Not LSM, but its sstable-based storage uses io_uring heavily
- **TigerBeetle** — Custom LSM-like storage, io_uring from day one
- **ClickHouse** — IOUringReader for async reads (not LSM, but columnar)

The pattern: C/C++/Zig/Rust implementations can adopt io_uring. Go implementations can't without significant cgo overhead.
