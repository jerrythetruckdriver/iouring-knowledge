# io_uring in Distributed Storage: Ceph, MinIO, and Others

## Ceph (BlueStore OSD)

Ceph has had io_uring support since 2019 as an alternative block I/O backend for BlueStore OSDs.

### Source Analysis

From `src/blk/kernel/io_uring.cc` (March 2026):

```cpp
struct ioring_data {
  struct io_uring io_uring;
  pthread_mutex_t cq_mutex;
  pthread_mutex_t sq_mutex;
  int epoll_fd = -1;
  std::map<int, int> fixed_fds_map;
};
```

Key design decisions:
- **Registered files** — All block device fds are registered upfront via `io_uring_register_files()`. Every SQE uses `IOSQE_FIXED_FILE`.
- **Mutex-protected submission and completion** — `sq_mutex` for submission, `cq_mutex` for reaping. Not ideal for performance but simple.
- **epoll bridging** — Uses epoll_wait on the ring fd to block for completions. Classic hybrid pattern.
- **Operations** — Only `readv` and `writev`. No fsync via io_uring (handled separately).
- **Optional IOPOLL and SQPOLL** — Configurable via `hipri` and `sq_thread` flags.

### What's Good

- Registered files eliminate fd lookup overhead per I/O
- IOPOLL mode for NVMe direct polling
- Simple, auditable code (~250 lines)

### What's Missing

- Mutexes on both SQ and CQ. A thread-per-core design with per-thread rings would be better.
- No registered buffers. Every I/O copies iovec pointers.
- No linked operations (could chain write+fsync).
- No provided buffer rings.
- epoll bridge adds an unnecessary syscall for completion waiting (could use `io_uring_enter` directly).

### Configuration

Enable in `ceph.conf`:
```ini
[osd]
bdev_ioring = true
bdev_ioring_hipri = true    # IOPOLL for NVMe
bdev_ioring_sqthread = true # SQPOLL thread
```

The backend is compiled only when `HAVE_LIBURING` is defined. Falls back to linux-aio otherwise.

## MinIO

MinIO is written in Go. No io_uring.

Go's runtime uses epoll for all I/O. The goroutine scheduler parks goroutines on epoll and unparks them on completion. There's no path to io_uring without either:
- cgo (defeats Go's concurrency model)
- A pure Go io_uring implementation (exists but immature, same goroutine impedance mismatch as io_uring-go)

MinIO recommends Linux kernel ≥ 6.8 for production but for filesystem/driver improvements, not io_uring.

## RocksDB / LevelDB

Neither uses io_uring. RocksDB uses `pread()`/`pwrite()` with its own thread pool for async I/O, plus `posix_fadvise()` for read-ahead.

There have been discussions about io_uring integration for RocksDB but nothing merged. The challenge: RocksDB's `Env` abstraction was designed around synchronous POSIX I/O. Retrofitting completion-based I/O requires rethinking the entire `RandomAccessFile`/`WritableFile` interface.

Ironically, applications using RocksDB over io_uring (like ScyllaDB) do so by bypassing RocksDB's I/O layer entirely for the hot path.

## FoundationDB

No io_uring. Uses its own `flow` actor system with `fdatasync()` for WAL and standard POSIX for data files. The actor model could theoretically map to io_uring's completion model but no work has been done.

## TigerBeetle

Full io_uring. Covered in [tigerbeetle.md](tigerbeetle.md) and [tigerbeetle-source.md](tigerbeetle-source.md). The gold standard for io_uring adoption in storage.

## Adoption Pattern

| Storage System | io_uring | Backend | Notes |
|----------------|----------|---------|-------|
| Ceph BlueStore | Optional | liburing, basic ops | Registered files, mutex-protected |
| ScyllaDB | Yes | Seastar io_uring | Thread-per-core, IOPOLL |
| TigerBeetle | Yes | Direct syscalls (Zig) | Three-queue architecture |
| MinIO | No | Go runtime (epoll) | Language limitation |
| RocksDB | No | pread/pwrite + threadpool | Interface mismatch |
| FoundationDB | No | POSIX + actor system | No work in progress |
| CockroachDB | No | Go runtime (epoll) | Language limitation |
| etcd | No | Go runtime (epoll) | Language limitation |

The pattern is clear: **systems written in C/C++/Zig/Rust adopt io_uring. Go systems can't.** And even among C/C++ systems, adoption requires rearchitecting the I/O layer — bolting io_uring onto an existing POSIX I/O abstraction yields minimal gains (see: Ceph's mutex approach).

## Sources

- Ceph `src/blk/kernel/io_uring.cc` (March 2026)
- Ceph `src/blk/kernel/io_uring.h` (March 2026)
- MinIO documentation (March 2026)
