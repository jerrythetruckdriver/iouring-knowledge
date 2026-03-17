# IORING_SETUP_NO_SQARRAY and SQ_REWIND

## The Problem: SQ Indirection

The original io_uring design includes an SQ index array — an indirection layer between the SQ ring and the SQE array. The idea was to allow non-contiguous SQE submission, letting userspace fill SQEs out of order.

In practice, almost nobody uses this. Every real implementation fills SQEs sequentially: get next SQE, fill it, bump the tail. The indirection array wastes a cache line per submission for zero benefit.

## NO_SQARRAY (kernel 6.7+)

```c
#define IORING_SETUP_NO_SQARRAY  (1U << 16)
```

Removes the SQ index array entirely. SQE index is derived directly from the SQ ring head/tail, same as CQEs work. One fewer memory region to mmap, one fewer indirection per submit.

**Impact:** Modest — saves one load per SQE submission. Matters at millions of ops/sec where every cache miss counts.

**liburing:** `io_uring_queue_init_params()` with `IORING_SETUP_NO_SQARRAY` in flags. liburing handles the rest transparently.

## SQ_REWIND (kernel 6.15+)

```c
#define IORING_SETUP_SQ_REWIND  (1U << 20)
```

Takes NO_SQARRAY further. With SQ_REWIND:
- Kernel ignores the SQ head/tail pointers entirely
- SQEs are fetched starting from index 0 every submission
- Application places all SQEs at the beginning of the SQ array, then calls `io_uring_enter()`

**Requirements:**
- Must also set `IORING_SETUP_NO_SQARRAY`
- Incompatible with `IORING_SETUP_SQPOLL` (kernel polls the ring — needs head/tail)
- SQ head and tail must remain 0

**Use case:** Batch-oriented workloads where you build a set of SQEs, submit them all, wait for completions, then build the next batch. No ring pointer management. Fill from 0, submit, repeat.

**When to use:**
- Fixed batch sizes (e.g., database WAL flushes, batch network sends)
- Simplified submission logic — no wrapping, no tail management
- NOT for streaming workloads where you continuously add SQEs

## Together

```c
struct io_uring_params params = {
    .flags = IORING_SETUP_NO_SQARRAY | IORING_SETUP_SQ_REWIND
             | IORING_SETUP_SINGLE_ISSUER | IORING_SETUP_DEFER_TASKRUN,
};
```

This is the simplest possible submission model: write SQEs at index 0..N-1, call enter, process completions, repeat. The ring is effectively just a flat array.
