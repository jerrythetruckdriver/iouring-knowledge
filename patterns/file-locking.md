# io_uring and File Locking

## Current State: No Async Locking Ops

There is no `IORING_OP_FLOCK` or `IORING_OP_FCNTL`. File locking via io_uring is not directly supported as of kernel 6.19.

## What Happens When You Lock

If you hold a POSIX lock (`fcntl(F_SETLK)`) or flock lock on a file, io_uring reads and writes to that file **ignore the lock entirely**. io_uring I/O operations bypass the locking checks that `read()`/`write()` syscalls perform.

This isn't a bug — it's consistent with how O_DIRECT and AIO also interact with locks. Locks are advisory by default on Linux.

## Workarounds

### 1. Lock Before Submitting I/O

```c
// Lock in the calling thread (blocking)
struct flock fl = { .l_type = F_WRLCK, .l_whence = SEEK_SET,
                    .l_start = offset, .l_len = length };
fcntl(fd, F_SETLKW, &fl);

// Now submit io_uring operations on the locked region
io_uring_prep_write(sqe, fd, buf, length, offset);
io_uring_submit(&ring);
// ... wait for CQE ...

// Unlock
fl.l_type = F_UNLCK;
fcntl(fd, F_SETLK, &fl);
```

Problem: The lock/unlock is synchronous. Fine if contention is rare.

### 2. Futex-Based Userspace Locking

Use `IORING_OP_FUTEX_WAIT`/`WAKE` (6.7+) for async userspace locks:

```c
// Userspace mutex backed by futex
atomic_int file_region_lock = 0;

// Try lock
if (!atomic_compare_exchange(&file_region_lock, &(int){0}, 1)) {
    // Contended — async wait via io_uring
    io_uring_prep_futex_wait(sqe, &file_region_lock, 1,
                             FUTEX_BITSET_MATCH_ANY, FUTEX2_SIZE_U32, 0);
    // When CQE completes, retry CAS
}

// After I/O completes, unlock + wake
atomic_store(&file_region_lock, 0);
io_uring_prep_futex_wake(sqe, &file_region_lock, 1,
                         FUTEX_BITSET_MATCH_ANY, FUTEX2_SIZE_U32, 0);
```

This only works for coordinating between threads/processes that agree on the locking protocol. Not interoperable with POSIX file locks.

### 3. O_APPEND for Write Serialization

If you just need serialized appends (logs, WAL):

```c
int fd = open("log", O_WRONLY | O_APPEND);
io_uring_prep_write(sqe, fd, buf, len, 0);  // offset ignored with O_APPEND
```

Kernel guarantees atomic append positioning. No locking needed.

### 4. Linked Ops as Serialization

For sequential operations that must not interleave:

```c
io_uring_prep_write(sqe1, fd, buf1, len1, off1);
sqe1->flags |= IOSQE_IO_LINK;
io_uring_prep_write(sqe2, fd, buf2, len2, off2);
sqe2->flags |= IOSQE_IO_LINK;
io_uring_prep_fsync(sqe3, fd, IORING_FSYNC_DATASYNC);
```

Linked ops execute in order. Not a lock, but provides ordered, atomic-ish sequences on a single ring.

## Mandatory Locks

Linux deprecated mandatory file locking (removed in 5.15). Don't rely on them.

## Bottom Line

io_uring doesn't do file locking. If your workload needs coordinated file access:
- Single-writer architectures (thread-per-core, one ring per file) eliminate the need
- Futex ops provide async userspace synchronization
- Advisory locks work if you lock synchronously before submitting I/O
- Linked ops provide in-ring ordering

The right answer is usually architectural: design so you don't need cross-thread file locks in the first place.
