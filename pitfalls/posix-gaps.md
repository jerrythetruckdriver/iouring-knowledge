# io_uring vs POSIX: Behavior Gaps

## Why This Matters

io_uring reimplements syscall semantics in an async context. Most of the time it behaves identically to the synchronous equivalent. But there are gaps, some by design, some by necessity.

## Known Divergences

### 1. Short Reads/Writes

**POSIX**: `read()` can return less than requested. `write()` can write less than requested. Application must retry.

**io_uring**: Same behavior. CQE `res` can be less than `len`. But many developers assume completion = full transfer. It's not.

**Fix**: Always check `cqe->res` against requested length. Resubmit for the remainder.

### 2. EINTR Handling

**POSIX**: Syscalls can return `-EINTR` when interrupted by a signal. Application retries.

**io_uring**: Requests are **automatically restarted** on EINTR. You will never see `-EINTR` in a CQE (with rare exceptions for some timeout operations). This is actually better behavior.

### 3. Error Semantics

**POSIX**: Errors are returned via `errno` (positive values, negated internally).

**io_uring**: CQE `res` contains **negative errno values** directly. `-EAGAIN`, not `EAGAIN`. No errno variable.

### 4. O_NONBLOCK

**POSIX**: File descriptors can be O_NONBLOCK, causing immediate EAGAIN on would-block.

**io_uring**: Ignores O_NONBLOCK for most operations. Requests are async by nature — they complete when ready, never return EAGAIN for "not ready yet." If you set IOSQE_ASYNC, the request always goes to the async worker pool.

**Exception**: Submissions can fail with `-EAGAIN` if the SQ is full or internal resources are exhausted.

### 5. close() Semantics

**POSIX**: `close(fd)` releases the fd immediately. The fd number can be reused before close() returns (by another thread).

**io_uring**: `IORING_OP_CLOSE` is async. The fd might still be "in use" by other in-flight io_uring requests on the same ring. Closing an fd while other requests reference it can cause those requests to fail with `-EBADF` or complete on the wrong fd if the number is reused.

**Best practice**: Cancel or drain all requests on an fd before closing it.

### 6. fork() and Ring Inheritance

**POSIX**: File descriptors are inherited across fork().

**io_uring**: The ring fd is inherited, but **the ring itself is not safely usable from the child**. The ring's mmap'd memory regions are shared (copy-on-write), but the kernel's io_ring_ctx is tied to the parent's task. Using the ring from a forked child is undefined behavior.

**Fix**: Create new rings in child processes. Don't inherit.

### 7. Thread Safety

**POSIX**: Most syscalls are thread-safe by default.

**io_uring**: Submission and completion are designed for **single-producer, single-consumer**. Multiple threads submitting to the same ring concurrently requires external synchronization (or `IORING_SETUP_SQPOLL` which has its own thread). SINGLE_ISSUER (6.1) enforces single-thread submission.

### 8. Signal Delivery

**POSIX**: Signals can interrupt syscalls at any point.

**io_uring**: `io_uring_enter()` accepts a signal mask via `IORING_ENTER_EXT_ARG`. But signals don't interrupt in-flight io_uring requests — they complete or fail on their own terms. No `SA_RESTART` equivalent needed.

### 9. fcntl/ioctl

**POSIX**: `fcntl()` and `ioctl()` are synchronous operations on file descriptors.

**io_uring**: No `IORING_OP_FCNTL` or `IORING_OP_IOCTL`. Some ioctl-like operations are available via `IORING_OP_URING_CMD` for specific drivers (NVMe, sockets), but there's no generic async fcntl/ioctl.

### 10. readv/writev Atomicity

**POSIX**: `readv()`/`writev()` are atomic with respect to the file offset on regular files.

**io_uring**: Same guarantee for `READV`/`WRITEV` on regular files. But with linked operations or concurrent submissions to the same file, ordering depends on whether requests are processed inline or via io-wq.

### 11. Timeout Resolution

**POSIX**: `select()`/`poll()` timeout resolution is implementation-defined (typically millisecond).

**io_uring**: Timeouts use `struct __kernel_timespec` (nanosecond resolution). With `IORING_REGISTER_CLOCK`, you can use BOOTTIME or REALTIME. The `min_wait_usec` field in registered wait provides microsecond batching control.

### 12. File Position (offset)

**POSIX**: `read()`/`write()` update the file position. `pread()`/`pwrite()` don't.

**io_uring**: `READ`/`WRITE` with `off = -1` uses and updates the current file position (like read/write). With `off >= 0`, it's positional (like pread/pwrite). `IORING_FEAT_RW_CUR_POS` indicates this support.

## Summary Table

| Behavior | POSIX | io_uring |
|---|---|---|
| EINTR | Returned to user | Auto-restarted |
| O_NONBLOCK | Honored | Mostly ignored |
| Error values | errno | Negative in CQE res |
| close + inflight | Immediate | Race condition possible |
| fork | FDs inherited | Ring not safely inheritable |
| Thread safety | Per-syscall | Requires SINGLE_ISSUER or sync |
| Signal interaction | Interrupts syscalls | Doesn't interrupt requests |
| fcntl/ioctl | Available | No generic op |

## The Pattern

io_uring is not "POSIX but async." It's a new I/O model that happens to implement many POSIX-like operations. Treat it as its own API with its own semantics. Read the CQE contract for each operation. Don't assume POSIX behavior holds.
