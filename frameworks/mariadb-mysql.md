# MariaDB / MySQL

## Status: No io_uring

Neither MariaDB nor MySQL uses io_uring.

## MySQL (InnoDB)

MySQL's InnoDB storage engine uses **Linux Native AIO** (`libaio`) for async I/O:

```c
// storage/innobase/os/os0file.cc
#ifdef LINUX_NATIVE_AIO
#include <libaio.h>
#endif
```

The AIO implementation handles:
- `io_submit()` for reads/writes
- `io_getevents()` for completions
- Thread-pool based fallback when native AIO unavailable

No io_uring code, no discussions in progress. MySQL's I/O layer is deeply tied to the libaio model.

## MariaDB (InnoDB)

MariaDB forked InnoDB and moved to a **thread pool** (`tpool`) architecture:

```c
// storage/innobase/os/os0file.cc
class io_slots {
    tpool::cache<tpool::aiocb, true> m_cache;
    tpool::task_group m_group;
};
```

Uses `tpool::aiocb` for async I/O with callback-based completions. Cross-platform, no Linux-specific optimizations.

## Why No Adoption

1. **Working AIO**: libaio does the job for O_DIRECT database workloads. Migration cost outweighs benefit.
2. **Cross-platform**: MySQL/MariaDB run on Linux, Windows, macOS, FreeBSD. io_uring is Linux-only.
3. **Buffer pool architecture**: InnoDB's buffer pool does its own caching. The I/O path is pread/pwrite with O_DIRECT — io_uring's batching wins are modest here.
4. **WAL path**: InnoDB's redo log is sequential write + fsync. Linked write+fsync in io_uring would help, but the log writer is already heavily optimized.

## Comparison with io_uring Adopters

| Database | I/O Backend | Why |
|---|---|---|
| MySQL | libaio | Working, cross-platform priority |
| MariaDB | tpool | Working, cross-platform priority |
| PostgreSQL | Sync I/O (io_uring WIP) | Andres Freund's async I/O patchset |
| RocksDB | **io_uring** | MultiRead batch optimization |
| ScyllaDB | **io_uring** | Thread-per-core, Linux-only |
| TigerBeetle | **io_uring** | Linux-only, designed around it |
| ClickHouse | **io_uring** | IOUringReader for async reads |

The pattern: databases that adopted io_uring are either Linux-only or have specific batch I/O patterns (RocksDB MultiRead, ScyllaDB per-shard rings) where io_uring's advantages are clear. Cross-platform databases stick with libaio or sync I/O.

## Percona Server

Percona Server for MySQL (enhanced MySQL fork) also uses libaio. No io_uring. Same codebase constraints.
