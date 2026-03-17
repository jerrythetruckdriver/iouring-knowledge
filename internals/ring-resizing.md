# Ring Resizing (IORING_REGISTER_RESIZE_RINGS)

*Kernel 6.13+*

## The Problem

Before 6.13, ring size was fixed at creation. Pick too small? You hit SQ/CQ full conditions under load. Pick too large? You waste memory on idle rings. Applications had to guess their peak workload upfront.

## The Solution

`IORING_REGISTER_RESIZE_RINGS` (opcode 33) resizes both SQ and CQ rings at runtime.

```c
struct io_uring_params p = {
    .sq_entries = new_sq_size,
    .cq_entries = new_cq_size,
};
io_uring_register(ring_fd, IORING_REGISTER_RESIZE_RINGS, &p, sizeof(p));
```

## Recommended Pattern

Start small, grow as needed:

```c
// Start with 64 entries
io_uring_queue_init(64, &ring, 0);

// Under load, grow to 4096
// (must drain pending work first)
```

This is now the officially recommended approach per Jens Axboe's pull request notes.

## Constraints

- Both SQ and CQ are resized together in one call
- The ring must be quiesced — no pending submissions during resize
- New sizes must be powers of 2
- The operation replaces the ring memory — mmap offsets change
- liburing handles re-mapping transparently

## Why Not Just Start Big?

Each SQE is 64 bytes (or 128 with SQE128). Each CQE is 16 bytes (or 32 with CQE32). A ring with 32K entries:
- SQ: 32K × 64B = 2MB
- CQ: 32K × 16B = 512KB

For applications that create many rings (one per thread, one per connection), memory adds up fast. Starting at 64 entries and growing to 4K only when needed saves ~90% memory in the common case.

## liburing Support

liburing wraps this with `io_uring_resize_rings()`. It handles the mmap dance — unmap old rings, update internal pointers to new rings.
