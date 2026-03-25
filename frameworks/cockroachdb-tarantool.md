# CockroachDB & Tarantool

Two databases. Neither uses io_uring. Different reasons.

## CockroachDB

**Language:** Go
**Storage engine:** Pebble (Go LSM, CockroachDB's fork of LevelDB concepts)
**I/O model:** Standard Go file I/O via `os.File`

CockroachDB can't use io_uring for the same reason every Go program can't: the Go runtime manages goroutine scheduling around blocking I/O via netpoller (epoll) and thread parking. There's no hook point for io_uring.

Pebble (their storage engine) is pure Go. Unlike RocksDB (C++, has io_uring for MultiRead), Pebble uses standard `pread`/`pwrite` syscalls. No cgo, no native io_uring path.

**Could they benefit?** Yes. Pebble's random SST reads during compaction and point lookups are exactly the workload where io_uring shines (batch multiple pread calls). But the Go runtime makes it impractical without a cgo wrapper, and the CockroachDB team has explicitly avoided cgo for build/deploy simplicity.

**RocksDB comparison:** RocksDB (C++) has production io_uring with thread-local rings, `SINGLE_ISSUER` + `DEFER_TASKRUN`, 256 depth. Used for `MultiRead` (batch SST random reads) and `ReadAsync` (prefetch). CockroachDB left RocksDB partly to avoid C/cgo dependencies.

## Tarantool

**Language:** C (core) + Lua (application logic)
**I/O model:** libev event loop with cooperative coroutines (fibers)
**Network:** epoll via libev
**Disk:** WAL writes via coio (coroutine I/O wrappers around libev)

Tarantool uses `ev_io`, `ev_stat`, `ev_timer` — the full libev API. Their fiber scheduler yields on I/O events from libev's epoll backend.

**Why no io_uring:**
1. Cross-platform: Tarantool runs on macOS, FreeBSD. libev abstracts this.
2. In-memory first: Tarantool is primarily an in-memory database. Disk I/O (WAL, snapshots) isn't the bottleneck.
3. Event loop lock-in: Replacing libev with io_uring would require rewriting the entire fiber scheduler.

**Where it could help:** WAL write+fsync path. Tarantool's WAL writer is a dedicated thread doing synchronous writes. A linked `WRITE + FSYNC` via io_uring would eliminate the thread and reduce latency. But the ROI doesn't justify the porting effort for an in-memory database.

## Pattern

Both follow the same pattern as most databases that don't use io_uring:

| Database | Language | Reason |
|----------|----------|--------|
| CockroachDB | Go | Go runtime, no cgo |
| Tarantool | C + Lua | libev, in-memory focus, cross-platform |
| PostgreSQL | C | In progress (Andres Freund's async I/O work) |
| MySQL/MariaDB | C++ | Cross-platform, libaio is "good enough" |
| MongoDB | C++ | Cross-platform (macOS/Windows), WiredTiger engine |

The databases that adopted io_uring (ScyllaDB, TigerBeetle, RocksDB, ClickHouse) share traits: Linux-only or Linux-primary, C/C++/Zig/Rust, and I/O-bound workloads.
