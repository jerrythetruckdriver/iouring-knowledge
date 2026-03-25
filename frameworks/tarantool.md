# Tarantool

**Type:** In-memory database + application server
**Language:** C (core) + Lua (application)
**I/O model:** libev event loop with cooperative fibers
**io_uring:** Not used

## Architecture

Tarantool is built on libev. The entire I/O stack uses `ev_io`, `ev_timer`, `ev_stat` watchers. Fibers (green threads) yield at I/O points, and libev's epoll backend dispatches events.

```
Application (Lua) → Fiber scheduler → coio wrappers → libev → epoll
```

WAL writer is a dedicated thread doing synchronous `write()` + `fdatasync()`.

## Why No io_uring

1. **In-memory first.** Data lives in RAM. Disk I/O is WAL-only for durability. The bottleneck is never disk read throughput.
2. **libev dependency.** Fiber scheduling, timers, signal handling, child process management — all through libev. Replacing it would be a rewrite.
3. **Cross-platform.** Tarantool supports macOS, FreeBSD. libev abstracts the event backend.

## Where It Could Help

- **WAL write path:** Linked `WRITE + FSYNC` would eliminate the dedicated WAL thread
- **Snapshot creation:** Large sequential writes during checkpointing
- **Vinyl engine:** Tarantool's disk-based storage engine (LSM) does disk reads for cold data — batched `pread` via io_uring would reduce latency

## See Also

- [cockroachdb-tarantool.md](cockroachdb-tarantool.md) — Combined analysis with CockroachDB
