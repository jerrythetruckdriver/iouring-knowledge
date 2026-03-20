# CQ Overflow Handling

## The Problem

Completion queue has finite entries. If the kernel posts a CQE and the CQ ring is full, you have overflow.

Default behavior since `IORING_FEAT_NODROP` (5.5): kernel stores overflowed CQEs in an internal list. They're flushed to the CQ ring when space opens. The kernel sets `IORING_SQ_CQ_OVERFLOW` in `sq_ring->flags` to signal this condition.

## Detection

```c
// Check for overflow condition
if (io_uring_sq_ready(&ring) && 
    (*ring.sq.kflags & IORING_SQ_CQ_OVERFLOW)) {
    // CQ overflowed — drain completions immediately
}
```

Or just check `cq_ring->overflow` counter — it increments for each overflow event.

## Why It Happens

1. **Multishot operations**: one SQE generates unbounded CQEs (multishot accept, multishot recv). If you're not draining fast enough, CQ fills.
2. **Burst completions**: batch of I/O completes simultaneously.
3. **Undersized CQ ring**: default CQ = 2 × SQ. High-throughput workloads need more.

## Strategies

### 1. Size the CQ Ring Correctly

```c
struct io_uring_params p = {
    .flags = IORING_SETUP_CQSIZE,
    .cq_entries = 4096,  // Independent of SQ size
};
io_uring_queue_init_params(256, &ring, &p);
```

Rule of thumb: CQ should be 4-8× SQ for multishot workloads. Each multishot accept can generate hundreds of CQEs before you drain.

### 2. Drain Before Submitting

```c
void event_loop(struct io_uring *ring) {
    while (1) {
        // Always drain CQ first
        struct io_uring_cqe *cqe;
        unsigned head;
        io_uring_for_each_cqe(ring, head, cqe) {
            process_cqe(cqe);
        }
        io_uring_cq_advance(ring, count);

        // Then submit new work
        io_uring_submit_and_wait(ring, 1);
    }
}
```

### 3. CQE_SKIP for Fire-and-Forget

```c
sqe->flags |= IOSQE_CQE_SKIP_SUCCESS;
```

Operations that succeed don't post a CQE. Failed operations still post. This dramatically reduces CQ pressure for operations where you only care about errors (e.g., timer rearms, MSG_RING signals).

### 4. Bundle Operations

`IORING_RECVSEND_BUNDLE` (6.10): one CQE reports multiple buffer completions. Instead of N CQEs for N buffers received, one CQE reports the count and starting buffer ID.

CQ pressure reduction: N:1 for batch recv/send.

### 5. Runtime Ring Resize

```c
// Start small, grow when needed (6.13+)
struct io_uring_params p = { .cq_entries = 256 };
io_uring_queue_init_params(64, &ring, &p);

// Later, when load increases:
// IORING_REGISTER_RESIZE_RINGS to grow CQ to 4096
```

### 6. Monitor and Alert

```c
static uint32_t last_overflow = 0;

void check_overflow(struct io_uring *ring) {
    uint32_t overflow = *ring->cq.koverflow;
    if (overflow != last_overflow) {
        fprintf(stderr, "CQ overflow: %u events lost\n", 
                overflow - last_overflow);
        last_overflow = overflow;
    }
}
```

"Lost" is misleading — with NODROP, events aren't lost, they're delayed. But the overflow path is slower (kernel internal list allocation, flush on next enter).

## Overflow Internals

When CQ overflows:

1. Kernel allocates `struct io_overflow_cqe` (heap) to store the CQE
2. CQE is added to `ctx->cq_overflow_list`
3. `IORING_SQ_CQ_OVERFLOW` flag is set
4. Next `io_uring_enter()` or CQ advance flushes overflow list to CQ ring
5. Flag is cleared when overflow list is empty

Each overflow allocation: ~48 bytes kernel heap. Under sustained overflow, this becomes a memory leak until drained.

## Anti-Patterns

| Pattern | Problem |
|---------|---------|
| CQ = 2 × SQ with multishot | Will overflow under load |
| Not draining CQ between submits | Guarantees overflow in sustained workloads |
| Ignoring overflow counter | Silent performance degradation |
| Using NODROP as a crutch | Overflow path is slow, design it away |

## Decision Matrix

| Workload | CQ:SQ Ratio | Extra Measures |
|----------|-------------|----------------|
| Simple file I/O | 2:1 (default) | None needed |
| Network server, no multishot | 4:1 | CQE_SKIP for timers |
| Multishot accept + recv | 8:1 or higher | Bundle, CQE_SKIP, monitor overflow |
| High-frequency trading | 4:1 + registered wait | Drain on every wake |

The best overflow strategy is to never overflow. Size your rings, drain eagerly, skip CQEs you don't need.
