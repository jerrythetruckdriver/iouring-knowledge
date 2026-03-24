# RocksDB io_uring Integration (Updated)

RocksDB has **production io_uring support** for reads. The previous characterization (Session 1) of "no io_uring" was wrong. The earlier revert was for a different approach; current integration is mature and active.

## Status

- **In production**: `ROCKSDB_IOURING_PRESENT` compile flag, auto-detected via CMake
- **Runtime control**: `RocksDbIOUringEnable()` weak symbol — define to return false to disable
- **Setup flags**: `IORING_SETUP_SINGLE_ISSUER | IORING_SETUP_DEFER_TASKRUN`
- **Ring depth**: 256 (`kIoUringDepth`)
- **Thread model**: Thread-local rings (one per thread, thread-local storage)

## What Uses io_uring

### MultiRead (batch random reads)

The primary use case. SST block reads during point lookups and range scans.

```
MultiRead(FSReadRequest* reqs, size_t num_reqs)
  → Thread-local io_uring (CreateIOUring if first use)
  → Batch io_uring_prep_readv() for each request
  → io_uring_submit_and_wait() (synchronous from caller's perspective)
  → io_uring_for_each_cqe() to reap completions
  → Short read resubmission (adjust iov_base/iov_len)
  → Fallback to serial pread() if io_uring init fails
```

Key design:
- `wrap_cache` (unordered_set) tracks all inflight requests
- `resubmit_rq_list` handles partial reads
- `finished_len` tracks cumulative bytes across resubmissions
- Uses `io_uring_sq_space_left()` and `io_uring_sq_ready()` for flow control
- Handles EINTR/EAGAIN as retryable, other errors as terminal
- On terminal error: reaps submitted CQEs, destroys ring, falls back

### ReadAsync (async prefetch)

Single-read async submission for prefetch operations:

```
ReadAsync(FSReadRequest& req)
  → io_uring_prep_readv()
  → io_uring_submit()
  → Returns handle for later Poll()/AbortIO()
```

### Poll (async completion check)

```
Poll(io_handle)
  → io_uring_wait_cqe()
  → Match CQE to handle via user_data
```

### AbortIO (cancellation)

```
AbortIO(io_handle)
  → io_uring_prep_cancel()
  → io_uring_submit()
  → Wait for cancel CQE
```

## Ring Configuration

```cpp
inline struct io_uring* CreateIOUring() {
    struct io_uring* new_io_uring = new struct io_uring;
    unsigned int flags = 0;
    flags |= IORING_SETUP_SINGLE_ISSUER;
    flags |= IORING_SETUP_DEFER_TASKRUN;
    int ret = io_uring_queue_init(kIoUringDepth, new_io_uring, flags);
    // ...
}
```

- **SINGLE_ISSUER**: Thread-local, no cross-thread submission
- **DEFER_TASKRUN**: Task work in submitting thread (no async task_work)
- **256 depth**: Sufficient for SST block read parallelism
- **Two rings per thread**: separate for MultiRead and ReadAsync

## What's NOT Using io_uring

- **Writes** — WAL writes, SST writes, compaction output (all synchronous pwrite)
- **Fsync** — fdatasync/fsync calls (could benefit from linked write+fsync)
- **Networking** — N/A (RocksDB is a library, not a server)
- **Compaction reads** — Sequential, uses pread (io_uring MultiRead is for random access)

## Error Handling

Thorough:
- EINTR/EAGAIN on submit: retry loop
- Terminal submit errors: reap already-submitted CQEs, destroy ring, recreate on next call
- CQE result < 0: IOError propagated per-request
- Short reads: automatic resubmission with adjusted offset/length
- User_data poisoning: `0xd5d5d5d5d5d5d5d5` written to consumed CQEs (debug aid)

## Compatibility

```cpp
// Compat defines in io_posix.h for older kernel headers
#ifndef IORING_SETUP_SINGLE_ISSUER
#define IORING_SETUP_SINGLE_ISSUER (1U << 12)
#endif
#ifndef IORING_SETUP_DEFER_TASKRUN
#define IORING_SETUP_DEFER_TASKRUN (1U << 13)
#endif
```

Falls back gracefully if io_uring init fails (old kernel, disabled, seccomp).

## Performance Impact

MultiRead with io_uring parallelizes what would be serial pread() calls:
- N SST block reads → 1 io_uring_submit_and_wait vs N pread syscalls
- Particularly beneficial with O_DIRECT on NVMe (multiple queue depths)
- SINGLE_ISSUER + DEFER_TASKRUN minimizes kernel overhead

## Assessment

RocksDB's integration is **conservative but correct**:
- Read-only (the easy, safe path)
- Thread-local rings (no sharing complexity)
- Proper error handling and fallback
- Modern setup flags (SINGLE_ISSUER, DEFER_TASKRUN)

Missing opportunity: linked write+fsync for WAL would reduce write latency. But writes aren't the bottleneck in most RocksDB workloads — random reads during compaction and point lookups are.

## Source

- `env/io_posix.h` — CreateIOUring(), kIoUringDepth, compat defines
- `env/io_posix.cc` — MultiRead, ReadAsync implementations
- `env/fs_posix.cc` — Poll, AbortIO, ring lifecycle
- `util/io_dispatcher_test.cc` — test with RocksDbIOUringEnable()
