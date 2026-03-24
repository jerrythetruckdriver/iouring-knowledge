# Accept Flags and Direct Descriptors

## accept4 Flags in io_uring

`IORING_OP_ACCEPT` supports standard accept4 flags via `sqe->accept_flags`:

```c
io_uring_prep_accept(sqe, listen_fd, addr, addrlen, SOCK_NONBLOCK | SOCK_CLOEXEC);
```

- **SOCK_NONBLOCK**: Set O_NONBLOCK on the new socket. With io_uring, you usually don't need this — io_uring handles async I/O regardless. But set it anyway if the fd might be used with epoll or traditional I/O paths.
- **SOCK_CLOEXEC**: Close on exec. Always set this unless you specifically need to inherit fds across exec.

## io_uring-Specific Accept Flags

Stored in `sqe->ioprio`:

```c
// Multishot accept (5.19)
sqe->ioprio |= IORING_ACCEPT_MULTISHOT;

// Don't wait if no pending connections
sqe->ioprio |= IORING_ACCEPT_DONTWAIT;

// Arm poll first, skip initial accept attempt (6.4)
sqe->ioprio |= IORING_ACCEPT_POLL_FIRST;
```

### IORING_ACCEPT_MULTISHOT

One SQE, unlimited CQEs. Each CQE delivers a new connection fd. CQE has `IORING_CQE_F_MORE` set until the accept terminates (error or cancellation).

### IORING_ACCEPT_DONTWAIT

Equivalent to MSG_DONTWAIT for accept. Returns -EAGAIN immediately if no pending connections. Useful for non-blocking accept checks without multishot.

### IORING_ACCEPT_POLL_FIRST

Skip the initial accept4 attempt, go straight to poll-arming. Reduces wasted work when there's no pending connection yet. Minor optimization for multishot accept.

## Direct Descriptors

The real power: accepted fds go directly into the fixed file table.

```c
io_uring_prep_multishot_accept_direct(sqe, listen_fd, NULL, NULL, 0);
// sqe->file_index = IORING_FILE_INDEX_ALLOC
```

With `IORING_FILE_INDEX_ALLOC`, the kernel picks the next available slot. The slot index is returned in `cqe->res`.

### Why This Matters

Normal accept: kernel allocates fd in process fd table → userspace gets fd number → register as fixed file → use fixed file. Three steps.

Direct accept: kernel allocates directly into fixed file table. One step. The fd never touches the process fd table. No `close()` needed — use `IORING_OP_CLOSE` with `IOSQE_FIXED_FILE`.

### Full Zero-Syscall Connection Setup

```
ACCEPT_DIRECT → [link] → SETSOCKOPT(TCP_NODELAY=1, fixed_file) → [multishot recv on fixed file]
```

From accept to first data receipt: zero syscalls. Everything through the ring.

## File Index Allocation

### Auto-Allocation

```c
sqe->file_index = IORING_FILE_INDEX_ALLOC;  // (~0U)
```

Kernel finds next free slot. Register allocation range first:

```c
struct io_uring_file_index_range range = { .off = 0, .len = 1024 };
io_uring_register(ring_fd, IORING_REGISTER_FILE_ALLOC_RANGE, &range, 0);
```

### Explicit Allocation

```c
sqe->file_index = 42 + 1;  // +1 because 0 means "use normal fd"
```

Pin to a specific slot. Useful when you want predictable fd-to-slot mapping.

## Common Patterns

### High-Connection Server

```c
// Register 65536 fixed file slots (sparse)
io_uring_register_files_sparse(ring, 65536);

// Set allocation range
struct io_uring_file_index_range range = { .off = 0, .len = 65536 };
io_uring_register(ring_fd, IORING_REGISTER_FILE_ALLOC_RANGE, &range, 0);

// Multishot accept into fixed table
io_uring_prep_multishot_accept_direct(sqe, listen_fd, NULL, NULL, 0);

// On each CQE: cqe->res is the fixed file index
// Start multishot recv on that index immediately
```

### Connection Limit

Auto-allocation returns -ENFILE when the fixed file table is full. Use this as natural backpressure — stop accepting when slots are exhausted.

## Gotchas

1. **file_index is 1-based for explicit allocation.** Value 0 means "use normal fd allocation." Slot N is `file_index = N + 1`.
2. **Auto-allocation is 0-based in CQE.** `cqe->res` returns the actual slot index (0-based). Asymmetric with SQE, but makes sense — CQE tells you where it went.
3. **Sparse registration required for auto-alloc.** You need `IORING_RSRC_REGISTER_SPARSE` or pre-register with -1 fds.
4. **Close must use IOSQE_FIXED_FILE.** Direct descriptors aren't in the process fd table. `close(cqe->res)` won't work.
