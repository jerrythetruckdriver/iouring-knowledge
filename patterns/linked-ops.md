# Linked Operations

Chain SQEs into dependency sequences. Next SQE executes only after previous completes.

## Link Types

### Soft Link (`IOSQE_IO_LINK`)
Chain breaks on failure. If any SQE fails, remaining linked SQEs are cancelled with `-ECANCELED`.

### Hard Link (`IOSQE_IO_HARDLINK`)
Chain continues regardless. Failed SQE doesn't cancel subsequent ones.

## Example: Read → Process → Write

```c
// Read
sqe = io_uring_get_sqe(ring);
io_uring_prep_read(sqe, src_fd, buf, len, 0);
sqe->flags |= IOSQE_IO_LINK;

// Write (only runs if read succeeds)
sqe = io_uring_get_sqe(ring);
io_uring_prep_write(sqe, dst_fd, buf, len, 0);
// No link flag on last SQE
```

## Link Timeout

Timeout for the entire chain:

```c
sqe = io_uring_get_sqe(ring);
io_uring_prep_read(sqe, fd, buf, len, 0);
sqe->flags |= IOSQE_IO_LINK;

sqe = io_uring_get_sqe(ring);
io_uring_prep_link_timeout(sqe, &ts, 0);
```

If the linked operation doesn't complete within the timeout, it's cancelled.

## Linked File Support (5.17+)

With `IORING_FEAT_LINKED_FILE`, linked SQEs can use the result fd from the previous SQE:

```c
// Open file
sqe = io_uring_get_sqe(ring);
io_uring_prep_openat(sqe, AT_FDCWD, path, O_RDONLY, 0);
sqe->flags |= IOSQE_IO_LINK;
sqe->file_index = IORING_FILE_INDEX_ALLOC;

// Read from opened file (uses allocated file index)
sqe = io_uring_get_sqe(ring);
io_uring_prep_read(sqe, 0, buf, len, 0); // fd filled from prev
sqe->flags |= IOSQE_FIXED_FILE;
```

## Drain

`IOSQE_IO_DRAIN` serializes an SQE with all previous SQEs. Everything before it must complete before it starts. Heavyweight — use links for targeted ordering.
