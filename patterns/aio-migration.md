# Migrating from linux-aio (libaio) to io_uring

## Why Migrate

| Feature | linux-aio (libaio) | io_uring |
|---------|-------------------|----------|
| File I/O | O_DIRECT only | Any file (buffered + direct) |
| Networking | No | Full TCP/UDP lifecycle |
| Polling | IOCB_FLAG_IOPOLL (limited) | IOPOLL, SQPOLL, hybrid |
| Buffer management | Per-iocb | Registered buffers, provided buffer rings |
| Linked operations | No | Chains, hard links, drain |
| Kernel versions | 2.6+ | 5.1+ (practical: 5.10+) |
| Syscalls per batch | 2 (io_submit + io_getevents) | 1 (io_uring_enter) or 0 (SQPOLL) |
| Max in-flight | Bounded by aio_max_nr sysctl | Bounded by ring size (resizable since 6.13) |
| Cancellation | io_cancel (unreliable) | ASYNC_CANCEL with 6 match modes |
| Completions | Copied to userspace buffer | Shared memory ring (zero-copy) |

## API Mapping

### Setup

```c
// libaio
io_context_t ctx;
io_setup(max_events, &ctx);

// io_uring
struct io_uring ring;
struct io_uring_params params = { .flags = IORING_SETUP_COOP_TASKRUN };
io_uring_queue_init_params(entries, &ring, &params);
```

### Submit Read

```c
// libaio
struct iocb cb;
io_prep_pread(&cb, fd, buf, len, offset);
cb.aio_data = (uintptr_t)user_data;
struct iocb *cbs[] = { &cb };
io_submit(ctx, 1, cbs);

// io_uring
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, fd, buf, len, offset);
io_uring_sqe_set_data(sqe, user_data);
io_uring_submit(&ring);
```

### Submit Write

```c
// libaio
struct iocb cb;
io_prep_pwrite(&cb, fd, buf, len, offset);
io_submit(ctx, 1, &(struct iocb*){ &cb });

// io_uring
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_write(sqe, fd, buf, len, offset);
io_uring_submit(&ring);
```

### Reap Completions

```c
// libaio
struct io_event events[max];
int n = io_getevents(ctx, min_nr, max, events, &timeout);
for (int i = 0; i < n; i++) {
    void *data = (void *)events[i].data;
    long res = events[i].res;
}

// io_uring
struct io_uring_cqe *cqe;
io_uring_wait_cqe(&ring, &cqe);  // or io_uring_peek_cqe for non-blocking
void *data = io_uring_cqe_get_data(cqe);
int res = cqe->res;
io_uring_cqe_seen(&ring, cqe);
```

### Vectored I/O

```c
// libaio
struct iocb cb;
io_prep_preadv(&cb, fd, iov, iovcnt, offset);

// io_uring
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_readv(sqe, fd, iov, iovcnt, offset);
```

### Teardown

```c
// libaio
io_destroy(ctx);

// io_uring
io_uring_queue_exit(&ring);
```

## Migration Strategy

### Phase 1: Drop-in Replacement

Replace libaio calls with io_uring equivalents. Keep the same event loop structure. This alone gets you:
- Buffered file I/O support (no more O_DIRECT requirement)
- Reliable cancellation
- Shared-memory completions (fewer copies)

### Phase 2: Register Resources

```c
// Register frequently-used fds
int fds[] = { fd1, fd2, fd3 };
io_uring_register_files(&ring, fds, 3);

// Register I/O buffers
struct iovec iovs[] = { { buf1, len1 }, { buf2, len2 } };
io_uring_register_buffers(&ring, iovs, 2);
```

Use `IOSQE_FIXED_FILE` and `io_uring_prep_read_fixed()`. Saves kernel fd/buffer lookups per I/O.

### Phase 3: Link Operations

What libaio can't do:

```c
// Atomic write + fsync chain
struct io_uring_sqe *sqe1 = io_uring_get_sqe(&ring);
io_uring_prep_write(sqe1, fd, buf, len, offset);
sqe1->flags |= IOSQE_IO_LINK;

struct io_uring_sqe *sqe2 = io_uring_get_sqe(&ring);
io_uring_prep_fsync(sqe2, fd, IORING_FSYNC_DATASYNC);

io_uring_submit(&ring);
// One submission, guaranteed order, one trip to kernel
```

### Phase 4: Advanced Features

- SQPOLL for zero-syscall submission
- IOPOLL for NVMe polling
- Provided buffer rings for async reads without pre-allocated buffers
- Timeout links for deadline-based I/O
- Batch submission of mixed operation types

## Common Gotchas

1. **Error semantics differ.** libaio's `io_event.res` and `io_event.res2` become a single `cqe->res`. Negative = errno.

2. **No implicit O_DIRECT requirement.** io_uring works with buffered I/O. Don't assume alignment requirements unless you're actually using O_DIRECT.

3. **CQE consumption is explicit.** Must call `io_uring_cqe_seen()` or `io_uring_cq_advance()`. Forgetting this stalls the ring.

4. **Ring size ≠ max in-flight.** SQ and CQ can have different sizes. CQ defaults to 2× SQ. Set `IORING_SETUP_CQSIZE` for explicit CQ sizing.

5. **aio_max_nr sysctl doesn't apply.** io_uring has its own limits via `RLIMIT_MEMLOCK` (for pinned pages) and ring entry counts.

## Who's Migrated

- **Ceph** — libaio → io_uring as alternative backend (optional)
- **QEMU** — linux-aio → io_uring as default block backend (since QEMU 5.0)
- **ScyllaDB** — linux-aio → io_uring via Seastar
- **fio** — Supports both, io_uring is the recommended engine for benchmarking

## Sources

- libaio API (`libaio.h`)
- liburing API reference
- Ceph `src/blk/kernel/` (aio vs io_uring backends)
- QEMU `block/io_uring.c` and `block/linux-aio.c`
