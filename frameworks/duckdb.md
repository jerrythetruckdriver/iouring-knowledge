# DuckDB and io_uring

## Status: Not Used

Zero io_uring references in the DuckDB codebase. Written in C++, cross-platform (Linux, macOS, Windows, WASM).

## Why Not

1. **Cross-platform priority** — DuckDB runs everywhere including WASM in browsers. Linux-only I/O isn't viable.
2. **Analytical workload** — OLAP queries are CPU-bound (hash joins, aggregations, vectorized execution). I/O is sequential scan with large reads, well-served by `pread()` + readahead.
3. **Embedded database** — runs in-process. No server, no event loop, no connection management. The use case that benefits most from io_uring (many concurrent connections, mixed I/O) doesn't apply.
4. **Memory-mapped I/O** — DuckDB uses mmap for buffer management on supported platforms. Page faults handle I/O implicitly.

## Where io_uring Could Theoretically Help

- **Parquet/CSV scanning** — parallel column chunk reads with batch submission
- **External table scans** — multiple file reads in parallel
- **Spill-to-disk** — hash join overflow writes

But DuckDB's vectorized execution engine already saturates memory bandwidth before I/O becomes the bottleneck on NVMe storage.

## The Pattern

Analytical databases care about compute throughput, not I/O syscall overhead. io_uring shines for OLTP workloads (many small random I/Os, connection management, fsync chains). DuckDB is the opposite: few large sequential I/Os, no connections, no durability requirements.

## See Also

- [ClickHouse](clickhouse.md) — analytical DB that DOES use io_uring (for async reads)
- [mmap vs io_uring](../benchmarks/mmap-vs-iouring.md) — when mmap wins
- [Database Patterns](../patterns/database-patterns.md) — OLTP vs OLAP I/O characteristics
