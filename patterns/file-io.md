# File I/O Patterns

## Basic Read/Write

```c
// Simple read
io_uring_prep_read(sqe, fd, buf, len, offset);

// Simple write
io_uring_prep_write(sqe, fd, buf, len, offset);
```

Use `offset = -1` to read/write at current file position (`IORING_FEAT_RW_CUR_POS` required).

## Registered Buffers

Pin buffers in kernel memory once, reference by index:

```c
// Register
struct iovec iovs[N];
io_uring_register_buffers(ring, iovs, N);

// Use
io_uring_prep_read_fixed(sqe, fd, buf, len, offset, buf_index);
```

Eliminates per-I/O `get_user_pages()`. Critical for NVMe/storage workloads where this dominates CPU time.

Vectored fixed I/O (6.14+):
```c
io_uring_prep_readv_fixed(sqe, fd, iovs, nr_iovs, offset, buf_index);
```

## Registered Files

```c
int fds[] = { fd1, fd2, fd3 };
io_uring_register_files(ring, fds, 3);

// Use file index instead of fd
sqe->fd = 0; // index into registered files
sqe->flags |= IOSQE_FIXED_FILE;
```

Saves `fget()`/`fput()` per operation. ~5% throughput improvement under contention.

Auto-allocated file slots:
```c
sqe->file_index = IORING_FILE_INDEX_ALLOC;
// cqe->res returns the allocated index
```

## O_DIRECT + IOPOLL

Maximum storage throughput:

```c
// Ring setup
params.flags = IORING_SETUP_IOPOLL;

// Open file
int fd = open(path, O_RDWR | O_DIRECT);

// Submit reads, then poll for completions
io_uring_submit(ring);
io_uring_wait_cqe(ring, &cqe); // polls instead of sleeping
```

`IOPOLL` only works with `O_DIRECT`. Kernel polls the block device for completions instead of waiting for IRQs.

## Read/Write Attributes (6.13+)

Attach metadata to I/O operations:

```c
struct io_uring_attr_pi pi = {
    .flags = ...,
    .app_tag = ...,
    .len = ...,
    .addr = ...,
    .seed = ...,
};

sqe->attr_ptr = (__u64)&pi;
sqe->attr_type_mask = IORING_RW_ATTR_FLAG_PI;
```

Protection information (T10-PI) for end-to-end data integrity.

## Multishot Read (6.6+)

Continuous read from an fd with provided buffers:

```c
io_uring_prep_read_multishot(sqe, fd, 0, 0, buf_group);
sqe->flags |= IOSQE_BUFFER_SELECT;
```

Each read produces a CQE with buffer ID. Keeps reading until error or cancellation.
