# io_uring Quick Reference / Cheat Sheet

## Ring Setup

```c
struct io_uring ring;

// Minimal
io_uring_queue_init(256, &ring, 0);

// Production (recommended flags for single-threaded)
io_uring_queue_init(256, &ring, 
    IORING_SETUP_COOP_TASKRUN | IORING_SETUP_SINGLE_ISSUER);

// High-throughput NVMe
io_uring_queue_init(4096, &ring,
    IORING_SETUP_IOPOLL | IORING_SETUP_SQPOLL);

// Cleanup
io_uring_queue_exit(&ring);
```

## Submit + Reap Pattern

```c
// Get SQE, prep, submit
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, fd, buf, len, offset);
io_uring_sqe_set_data(sqe, my_context);
io_uring_submit(&ring);

// Wait + reap
struct io_uring_cqe *cqe;
io_uring_wait_cqe(&ring, &cqe);
int result = cqe->res;          // bytes transferred or -errno
void *ctx = io_uring_cqe_get_data(cqe);
io_uring_cqe_seen(&ring, cqe);
```

## Batch Reap (Drain All)

```c
unsigned head;
struct io_uring_cqe *cqe;
unsigned count = 0;
io_uring_for_each_cqe(&ring, head, cqe) {
    process(cqe);
    count++;
}
io_uring_cq_advance(&ring, count);
```

## Common Operations

### File I/O
```c
io_uring_prep_read(sqe, fd, buf, len, offset);
io_uring_prep_write(sqe, fd, buf, len, offset);
io_uring_prep_readv(sqe, fd, iovs, nr_vecs, offset);
io_uring_prep_writev(sqe, fd, iovs, nr_vecs, offset);
io_uring_prep_fsync(sqe, fd, 0);                    // fsync
io_uring_prep_fsync(sqe, fd, IORING_FSYNC_DATASYNC); // fdatasync
io_uring_prep_openat(sqe, AT_FDCWD, path, flags, mode);
io_uring_prep_close(sqe, fd);
io_uring_prep_statx(sqe, AT_FDCWD, path, 0, STATX_ALL, statxbuf);
```

### Networking
```c
io_uring_prep_socket(sqe, AF_INET, SOCK_STREAM, 0, 0);
io_uring_prep_bind(sqe, fd, addr, addrlen);           // 6.14+
io_uring_prep_listen(sqe, fd, backlog);                // 6.14+
io_uring_prep_accept(sqe, fd, addr, addrlen, 0);
io_uring_prep_recv(sqe, fd, buf, len, 0);
io_uring_prep_send(sqe, fd, buf, len, 0);
io_uring_prep_connect(sqe, fd, addr, addrlen);
io_uring_prep_shutdown(sqe, fd, SHUT_WR);

// Multishot (CQE_F_MORE set until done)
io_uring_prep_multishot_accept(sqe, fd, NULL, NULL, 0);
io_uring_prep_recv_multishot(sqe, fd, NULL, 0, 0);    // + IOSQE_BUFFER_SELECT
```

### Timeouts
```c
struct __kernel_timespec ts = { .tv_sec = 5 };
io_uring_prep_timeout(sqe, &ts, 0, 0);               // Relative
io_uring_prep_timeout(sqe, &ts, 0, IORING_TIMEOUT_ABS); // Absolute
io_uring_prep_link_timeout(sqe, &ts, 0);              // Deadline for linked op
```

## SQE Flags

```c
sqe->flags |= IOSQE_FIXED_FILE;        // fd is index into registered files
sqe->flags |= IOSQE_IO_LINK;           // Link to next SQE (soft)
sqe->flags |= IOSQE_IO_HARDLINK;       // Link (continue on error)
sqe->flags |= IOSQE_IO_DRAIN;          // Wait for all prior SQEs
sqe->flags |= IOSQE_ASYNC;             // Force async (skip inline fast path)
sqe->flags |= IOSQE_BUFFER_SELECT;     // Use provided buffer ring
sqe->flags |= IOSQE_CQE_SKIP_SUCCESS;  // No CQE on success
```

## CQE Flags

```c
if (cqe->flags & IORING_CQE_F_MORE)       // Multishot: more CQEs coming
if (cqe->flags & IORING_CQE_F_BUFFER)     // Buffer ID in upper 16 bits
if (cqe->flags & IORING_CQE_F_NOTIF)      // Zero-copy send notification
if (cqe->flags & IORING_CQE_F_BUF_MORE)   // Incremental buffer consumption
if (cqe->flags & IORING_CQE_F_SKIP)       // Ignore this CQE (gap fill)

// Extract buffer ID
int buf_id = cqe->flags >> IORING_CQE_BUFFER_SHIFT;
```

## Registered Resources

```c
// Files
int fds[] = { fd1, fd2, fd3 };
io_uring_register_files(&ring, fds, 3);
// Then: sqe->flags |= IOSQE_FIXED_FILE; sqe->fd = index;

// Buffers
struct iovec iovs[] = { {buf1, len1}, {buf2, len2} };
io_uring_register_buffers(&ring, iovs, 2);
// Then: io_uring_prep_read_fixed(sqe, fd, buf, len, off, buf_index);

// Provided Buffer Ring
struct io_uring_buf_reg reg = {
    .ring_addr = (unsigned long)buf_ring,
    .ring_entries = 256,
    .bgid = 0,
};
io_uring_register_buf_ring(&ring, &reg, 0);
```

## Linked Operations

```c
// Write then fsync (atomic chain)
sqe1 = io_uring_get_sqe(&ring);
io_uring_prep_write(sqe1, fd, buf, len, off);
sqe1->flags |= IOSQE_IO_LINK;

sqe2 = io_uring_get_sqe(&ring);
io_uring_prep_fsync(sqe2, fd, IORING_FSYNC_DATASYNC);

io_uring_submit(&ring);
// sqe2 executes only if sqe1 succeeds (soft link)
// Use IOSQE_IO_HARDLINK to continue even on error
```

## Cancellation

```c
// Cancel by user_data
io_uring_prep_cancel(sqe, user_data, 0);

// Cancel by fd
io_uring_prep_cancel_fd(sqe, fd, IORING_ASYNC_CANCEL_FD);

// Cancel everything
io_uring_prep_cancel(sqe, 0, IORING_ASYNC_CANCEL_ANY | IORING_ASYNC_CANCEL_ALL);
```

## Feature Detection

```c
struct io_uring_probe *probe = io_uring_get_probe_ring(&ring);
if (io_uring_opcode_supported(probe, IORING_OP_RECV_ZC))
    // Zero-copy receive available
io_uring_free_probe(probe);
```

## Setup Flag Combos

| Goal | Flags |
|------|-------|
| Minimal overhead | `COOP_TASKRUN \| SINGLE_ISSUER` |
| Zero syscalls | `SQPOLL \| COOP_TASKRUN` |
| Max NVMe IOPS | `IOPOLL \| SQPOLL \| COOP_TASKRUN` |
| Latency-sensitive | `DEFER_TASKRUN \| SINGLE_ISSUER` |
| Memory-efficient | `NO_SQARRAY \| CQE_MIXED` (6.18+) |

## Kernel Version Requirements

| Feature | Minimum Kernel |
|---------|---------------|
| Basic io_uring | 5.1 |
| Provided buffers | 5.7 |
| Fixed files | 5.1 |
| SQPOLL | 5.1 |
| IOPOLL | 5.1 |
| Multishot accept | 5.19 |
| Multishot recv | 6.0 |
| Provided buffer rings | 5.19 |
| COOP_TASKRUN | 5.19 |
| SINGLE_ISSUER | 6.0 |
| DEFER_TASKRUN | 6.1 |
| Zero-copy send | 6.0 |
| Socket/bind/listen | 6.14 |
| Zero-copy recv (zcrx) | 6.15 |
| Ring resize | 6.13 |
| Mixed CQE sizes | 6.18 |
| REGISTER_QUERY | 6.19 |
