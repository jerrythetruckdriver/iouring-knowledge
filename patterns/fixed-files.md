# Fixed File Table Management

## Why Fixed Files

Every io_uring operation that takes an fd does `fget()`/`fput()` — atomic refcount operations on the file struct. Under heavy load, this causes cache line bouncing across cores. Fixed files (`IOSQE_FIXED_FILE`) skip the refcounting entirely by using a pre-registered table.

Measured impact: 5-15% throughput improvement on network-heavy workloads. More on NUMA systems where atomic ops cross sockets.

## Registration

### Static Registration

```c
int fds[64];
// fill fds with open file descriptors
io_uring_register_files(&ring, fds, 64);
```

All slots must be filled or set to -1 (empty). Use `IORING_RSRC_REGISTER_SPARSE` to avoid pre-filling:

```c
struct io_uring_rsrc_register reg = {
    .nr = 1024,
    .flags = IORING_RSRC_REGISTER_SPARSE,
};
io_uring_register_buffers2(&ring, &reg);  // all slots start empty
```

### Dynamic Updates

```c
// Install fd 42 at slot 10
io_uring_register_files_update(&ring, 10, &fd, 1);

// Remove slot 10
int minus_one = -1;
io_uring_register_files_update(&ring, 10, &minus_one, 1);
```

Updates are synchronous — they wait for in-flight operations on the old fd to complete.

### Auto-Allocation

For opcodes that create new fds (accept, openat, openat2, socket, pipe):

```c
sqe->file_index = IORING_FILE_INDEX_ALLOC;  // (~0U)
```

Kernel picks the next available slot. CQE returns the slot index in `cqe->res`, not a regular fd. The fd is never installed in the process fd table.

### Alloc Range

```c
struct io_uring_file_index_range range = {
    .off = 0,
    .len = 256,
};
io_uring_register(fd, IORING_REGISTER_FILE_ALLOC_RANGE, &range, 0);
```

Constrains auto-allocation to slots [off, off+len). Useful when you want to partition the fixed file table — e.g., slots 0-255 for accepted connections, 256+ for manually managed fds.

### OP_FIXED_FD_INSTALL

```c
// Export a fixed file slot to a regular fd (visible to non-io_uring code)
io_uring_prep_fixed_fd_install(sqe, fixed_slot, 0);
```

Returns a normal fd in cqe->res. Use when you need to pass the fd to code that doesn't know about io_uring fixed files (e.g., a library, another thread's event loop).

`IORING_FIXED_FD_NO_CLOEXEC` flag skips setting O_CLOEXEC on the installed fd.

## Patterns

### Connection Lifecycle (fully fixed)

```
socket(AF_INET, SOCK_STREAM) → fixed slot via IORING_FILE_INDEX_ALLOC
bind() via IORING_OP_BIND on fixed fd
listen() via IORING_OP_LISTEN on fixed fd
accept() → auto-alloc into fixed slot
recv/send on fixed fd
close via IORING_OP_CLOSE on fixed fd (frees the slot)
```

Zero process fd table involvement. Zero fget/fput overhead.

### Scaling

- Up to millions of registered files per ring
- Updates are O(1) per slot
- Memory: ~8 bytes per slot (pointer + RCU overhead)
- Clone buffers across rings with `IORING_REGISTER_CLONE_BUFFERS` (kernel 6.12+)

## Gotchas

- Can't use fixed files with SQPOLL until SQPOLL thread is running
- Closing a fixed file slot doesn't close the underlying fd — it just removes the registration
- `IORING_OP_CLOSE` with `IOSQE_FIXED_FILE` both unregisters and closes
- Fixed file index 0 is valid — don't confuse it with "not set"
- Thread safety: file table updates are serialized per ring
