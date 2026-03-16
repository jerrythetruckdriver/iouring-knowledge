# liburing API Reference

liburing is the userspace library for io_uring. You *can* use the raw syscalls directly, but you shouldn't.

Repository: [axboe/liburing](https://github.com/axboe/liburing)

## Ring Lifecycle

```c
// Initialize with default params
int io_uring_queue_init(unsigned entries, struct io_uring *ring, unsigned flags);

// Initialize with full params control
int io_uring_queue_init_params(unsigned entries, struct io_uring *ring,
                               struct io_uring_params *p);

// Teardown
void io_uring_queue_exit(struct io_uring *ring);
```

`entries` must be power of 2. `flags` maps to `IORING_SETUP_*`.

## SQE Prep Helpers

Every opcode has a `io_uring_prep_*` helper. These zero and fill the SQE. Never fill SQE fields manually — the struct layout changes.

### File I/O
```c
void io_uring_prep_read(struct io_uring_sqe *sqe, int fd, void *buf,
                        unsigned nbytes, __u64 offset);
void io_uring_prep_write(struct io_uring_sqe *sqe, int fd, const void *buf,
                         unsigned nbytes, __u64 offset);
void io_uring_prep_readv(struct io_uring_sqe *sqe, int fd,
                         const struct iovec *iovecs, unsigned nr_vecs,
                         __u64 offset);
void io_uring_prep_writev(struct io_uring_sqe *sqe, int fd,
                          const struct iovec *iovecs, unsigned nr_vecs,
                          __u64 offset);
void io_uring_prep_read_fixed(struct io_uring_sqe *sqe, int fd, void *buf,
                              unsigned nbytes, __u64 offset, int buf_index);
void io_uring_prep_write_fixed(struct io_uring_sqe *sqe, int fd,
                               const void *buf, unsigned nbytes,
                               __u64 offset, int buf_index);
```

### Networking
```c
void io_uring_prep_accept(struct io_uring_sqe *sqe, int fd,
                          struct sockaddr *addr, socklen_t *addrlen, int flags);
void io_uring_prep_multishot_accept(struct io_uring_sqe *sqe, int fd,
                                    struct sockaddr *addr, socklen_t *addrlen,
                                    int flags);
void io_uring_prep_connect(struct io_uring_sqe *sqe, int fd,
                           const struct sockaddr *addr, socklen_t addrlen);
void io_uring_prep_recv(struct io_uring_sqe *sqe, int fd, void *buf,
                        size_t len, int flags);
void io_uring_prep_recv_multishot(struct io_uring_sqe *sqe, int fd,
                                  void *buf, size_t len, int flags);
void io_uring_prep_send(struct io_uring_sqe *sqe, int fd, const void *buf,
                        size_t len, int flags);
void io_uring_prep_send_zc(struct io_uring_sqe *sqe, int fd, const void *buf,
                           size_t len, int flags, unsigned zc_flags);
void io_uring_prep_socket(struct io_uring_sqe *sqe, int domain, int type,
                          int protocol, unsigned flags);
void io_uring_prep_bind(struct io_uring_sqe *sqe, int fd,
                        struct sockaddr *addr, socklen_t addrlen);
void io_uring_prep_listen(struct io_uring_sqe *sqe, int fd, int backlog);
```

### Filesystem
```c
void io_uring_prep_openat(struct io_uring_sqe *sqe, int dfd,
                          const char *path, int flags, mode_t mode);
void io_uring_prep_close(struct io_uring_sqe *sqe, int fd);
void io_uring_prep_statx(struct io_uring_sqe *sqe, int dfd, const char *path,
                         int flags, unsigned mask, struct statx *statxbuf);
void io_uring_prep_renameat(struct io_uring_sqe *sqe, int olddfd,
                            const char *oldpath, int newdfd,
                            const char *newpath, int flags);
void io_uring_prep_unlinkat(struct io_uring_sqe *sqe, int dfd,
                            const char *path, int flags);
void io_uring_prep_mkdirat(struct io_uring_sqe *sqe, int dfd,
                           const char *path, mode_t mode);
void io_uring_prep_fsync(struct io_uring_sqe *sqe, int fd, unsigned flags);
```

### Control
```c
void io_uring_prep_cancel(struct io_uring_sqe *sqe, __u64 user_data,
                          int flags);
void io_uring_prep_timeout(struct io_uring_sqe *sqe,
                           struct __kernel_timespec *ts,
                           unsigned count, unsigned flags);
void io_uring_prep_link_timeout(struct io_uring_sqe *sqe,
                                struct __kernel_timespec *ts, unsigned flags);
void io_uring_prep_msg_ring(struct io_uring_sqe *sqe, int fd, unsigned len,
                            __u64 data, unsigned flags);
void io_uring_prep_nop(struct io_uring_sqe *sqe);
```

## Submission & Completion

```c
// Get next SQE slot (returns NULL if SQ is full)
struct io_uring_sqe *io_uring_get_sqe(struct io_uring *ring);

// Submit all queued SQEs
int io_uring_submit(struct io_uring *ring);

// Submit and wait for at least `wait_nr` completions
int io_uring_submit_and_wait(struct io_uring *ring, unsigned wait_nr);

// Wait for a CQE
int io_uring_wait_cqe(struct io_uring *ring, struct io_uring_cqe **cqe_ptr);

// Peek (non-blocking)
int io_uring_peek_cqe(struct io_uring *ring, struct io_uring_cqe **cqe_ptr);

// Batch peek
unsigned io_uring_peek_batch_cqe(struct io_uring *ring,
                                  struct io_uring_cqe **cqes, unsigned count);

// Mark CQE as consumed
void io_uring_cqe_seen(struct io_uring *ring, struct io_uring_cqe *cqe);

// Advance head by `nr` CQEs (batch version of cqe_seen)
void io_uring_cq_advance(struct io_uring *ring, unsigned nr);
```

## User Data

```c
// Set user_data on SQE
void io_uring_sqe_set_data(struct io_uring_sqe *sqe, void *data);
void io_uring_sqe_set_data64(struct io_uring_sqe *sqe, __u64 data);

// Get user_data from CQE
void *io_uring_cqe_get_data(const struct io_uring_cqe *cqe);
__u64 io_uring_cqe_get_data64(const struct io_uring_cqe *cqe);
```

## SQE Flags

```c
void io_uring_sqe_set_flags(struct io_uring_sqe *sqe, unsigned flags);
```

Flags: `IOSQE_FIXED_FILE`, `IOSQE_IO_DRAIN`, `IOSQE_IO_LINK`, `IOSQE_IO_HARDLINK`, `IOSQE_ASYNC`, `IOSQE_BUFFER_SELECT`, `IOSQE_CQE_SKIP_SUCCESS`.

## Registration

```c
int io_uring_register_buffers(struct io_uring *ring,
                              const struct iovec *iovecs, unsigned nr_iovecs);
int io_uring_unregister_buffers(struct io_uring *ring);

int io_uring_register_files(struct io_uring *ring,
                            const int *files, unsigned nr_files);
int io_uring_unregister_files(struct io_uring *ring);
int io_uring_register_files_update(struct io_uring *ring, unsigned off,
                                   const int *files, unsigned nr_files);

int io_uring_register_ring_fd(struct io_uring *ring);
int io_uring_unregister_ring_fd(struct io_uring *ring);
```

## Feature Probing

```c
// Check if an opcode is supported
struct io_uring_probe *io_uring_get_probe(void);
int io_uring_opcode_supported(const struct io_uring_probe *p, int op);
void io_uring_free_probe(struct io_uring_probe *probe);
```

Always probe before using newer opcodes. Don't assume kernel version = feature availability.

## Common Patterns

### Basic event loop
```c
struct io_uring ring;
io_uring_queue_init(256, &ring, 0);

while (running) {
    struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
    io_uring_prep_read(sqe, fd, buf, len, 0);
    io_uring_sqe_set_data64(sqe, MY_READ_TAG);
    io_uring_submit(&ring);

    struct io_uring_cqe *cqe;
    io_uring_wait_cqe(&ring, &cqe);
    // handle cqe->res
    io_uring_cqe_seen(&ring, cqe);
}

io_uring_queue_exit(&ring);
```

### Batch completion processing
```c
struct io_uring_cqe *cqes[32];
unsigned count = io_uring_peek_batch_cqe(&ring, cqes, 32);
for (unsigned i = 0; i < count; i++) {
    handle_completion(cqes[i]);
}
io_uring_cq_advance(&ring, count);
```
