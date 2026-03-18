# SQ and CQ Backpressure Patterns

## The Problem

Two pressure points in io_uring:
1. **SQ full** — Can't submit more SQEs. Ring is backed up.
2. **CQ overflow** — Kernel can't post CQEs. Completions are lost or deferred.

Both are your fault. Here's how to handle them.

## SQ Backpressure

### Detection

```c
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
if (!sqe) {
    // SQ is full. Must submit or wait.
}
```

`io_uring_get_sqe()` returns NULL when `sq_tail - sq_head == sq_entries`.

### Strategy 1: Submit and Retry

```c
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
if (!sqe) {
    io_uring_submit(&ring);  // Flush pending SQEs to kernel
    sqe = io_uring_get_sqe(&ring);
    assert(sqe);  // Should succeed now unless everything is in-flight
}
```

### Strategy 2: Drain CQ First

If SQ is full because everything is in-flight and the CQ is also backing up:

```c
while (!io_uring_get_sqe(&ring)) {
    struct io_uring_cqe *cqe;
    io_uring_submit_and_wait(&ring, 1);  // Submit + wait for at least 1 CQE
    
    unsigned head;
    io_uring_for_each_cqe(&ring, head, cqe) {
        process_completion(cqe);
    }
    io_uring_cq_advance(&ring, count);
}
```

### Strategy 3: Userspace Queue

Keep an overflow queue in userspace. When the SQ is full, park requests there and drain on next iteration.

```c
struct pending_request {
    int fd;
    void *buf;
    size_t len;
    off_t offset;
    struct pending_request *next;
};

struct pending_request *overflow_queue = NULL;

void submit_or_queue(int fd, void *buf, size_t len, off_t offset) {
    struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
    if (sqe) {
        io_uring_prep_read(sqe, fd, buf, len, offset);
    } else {
        // Park it
        enqueue(&overflow_queue, fd, buf, len, offset);
    }
}

// In event loop: drain overflow queue after each CQ reap
```

TigerBeetle does exactly this with its three-queue architecture (unqueued → SQ → completed).

### Strategy 4: Ring Resize (6.13+)

If backpressure is chronic, grow the ring at runtime:

```c
struct io_uring_params p = { .sq_entries = 4096, .cq_entries = 8192 };
io_uring_register_resize_rings(&ring, &p);
```

Start small (64–256), resize under load, shrink during idle.

## CQ Overflow

### How It Happens

Kernel posts a CQE but `cq_tail - cq_head == cq_entries`. The CQ is full because the application isn't consuming CQEs fast enough.

### Detection

```c
// Check SQ flags for overflow indication
uint32_t sq_flags = IO_URING_READ_ONCE(*ring->sq.kflags);
if (sq_flags & IORING_SQ_CQ_OVERFLOW) {
    // CQ has overflowed. Kernel is buffering internally.
}
```

### Default Behavior (IORING_FEAT_NODROP)

Modern kernels (5.5+) with `IORING_FEAT_NODROP`:
- Kernel keeps overflow CQEs in an internal list
- New submissions may block or return `-EBUSY` until overflow is drained
- `IORING_SQ_CQ_OVERFLOW` flag is set

Drain the CQ aggressively when you see this flag.

### CQE_SKIP to Reduce Pressure

For fire-and-forget operations, skip CQE generation:

```c
sqe->flags |= IOSQE_CQE_SKIP_SUCCESS;
```

No CQE posted on success. CQE still posted on error. Halves CQ pressure for operations where you don't need success confirmation.

Use cases:
- Buffer refills via PROVIDE_BUFFERS
- MSG_RING notifications
- Timer updates
- NOP heartbeats

### Size the CQ Correctly

```c
struct io_uring_params params = {
    .flags = IORING_SETUP_CQSIZE,
    .cq_entries = sq_entries * 4,  // 4x SQ for multishot headroom
};
```

Default is 2× SQ entries. If using multishot operations (accept, recv), each SQE generates multiple CQEs. Size accordingly.

## Batch Processing Pattern

The optimal event loop:

```c
while (running) {
    // 1. Submit everything pending
    io_uring_submit(&ring);
    
    // 2. Wait for at least 1 completion
    io_uring_wait_cqe(&ring, &cqe);
    
    // 3. Drain ALL available CQEs (not just one)
    unsigned head;
    unsigned count = 0;
    io_uring_for_each_cqe(&ring, head, cqe) {
        process_completion(cqe);
        count++;
    }
    io_uring_cq_advance(&ring, count);
    
    // 4. Refill SQ from overflow queue
    drain_overflow_queue();
}
```

Or use `io_uring_submit_and_wait()` to combine steps 1-2 into a single syscall.

### min_wait_usec Batching (6.13+)

```c
struct io_uring_reg_wait w = {
    .min_wait_usec = 100,  // Wait up to 100μs for more completions
    .ts = { .tv_sec = 1 }, // Absolute timeout
};
```

Trades latency for throughput. Kernel batches completions during the wait window.

## Anti-Patterns

- **Submitting one SQE at a time.** Batch. Always batch.
- **Processing one CQE per wait.** Drain the entire CQ each iteration.
- **Ignoring `IORING_SQ_CQ_OVERFLOW`.** This means you're losing (or deferring) completions.
- **SQ size == CQ size with multishot.** CQ will overflow because multishot generates N CQEs per SQE.
- **No overflow queue.** When the SQ fills, you need somewhere to park requests. Dropping them is not a plan.
