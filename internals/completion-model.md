# CQE Delivery Model

How completions get from kernel to userspace. Understanding this is key to writing correct io_uring code.

## Completion Paths

A request can complete through three paths:

### 1. Inline Completion

Request completes during `io_uring_enter()` or SQPOLL submission. The CQE is posted immediately. No task work, no IPI. This is the fast path.

Happens when:
- Data is in page cache (cached read)
- Socket send buffer has room (small send)
- Operation is trivial (NOP, MSG_RING)

### 2. Async Completion via Task Work

Request goes async (poll-armed or io-wq). When it completes, the kernel posts a CQE and schedules task work on the submitting task. The task processes completions on next kernel transition.

With `IORING_SETUP_COOP_TASKRUN`: task checks for work on voluntary kernel transitions (syscalls). No IPI interrupts.

With `IORING_SETUP_DEFER_TASKRUN`: task work deferred until `io_uring_enter()` with `IORING_ENTER_GETEVENTS`. Maximum batching.

### 3. SQPOLL Thread Completion

SQPOLL kernel thread handles both submission and inline completions. CQEs appear in the CQ ring without the application making any syscall. The application just polls the CQ tail.

## CQ Ring Mechanics

The CQ ring is a shared-memory SPSC (single-producer, single-consumer) ring buffer.

**Producer:** Kernel writes CQEs and advances `cq->tail`.
**Consumer:** Application reads CQEs and advances `cq->head`.

Memory ordering:
```
Kernel (producer):                    Userspace (consumer):
  write CQE fields                     read cq->tail (acquire)
  smp_store_release(&cq->tail, new)    read CQE fields
                                        smp_store_release(&cq->head, new)
```

On x86 (TSO): store-release is just a compiler barrier. On ARM: full DMB/DSB barriers.

## CQE Visibility

A CQE is visible to userspace only after the kernel advances `cq->tail` with a store-release. The application must read `cq->tail` with a load-acquire before reading CQE fields. liburing handles this:

```c
// liburing does the right thing internally
io_uring_peek_cqe(&ring, &cqe);  // load-acquire on tail
// cqe->res and cqe->user_data are now safe to read
```

## Batch Completion

Multiple requests can complete in a single batch:

```c
// Process all available CQEs without waiting
unsigned head;
struct io_uring_cqe *cqe;
unsigned count = 0;

io_uring_for_each_cqe(&ring, head, cqe) {
    // Process cqe
    count++;
}
io_uring_cq_advance(&ring, count);  // single store-release for all
```

`io_uring_cq_advance()` advances the head by N in one store — more efficient than calling `io_uring_cqe_seen()` N times (which does N stores).

## CQE Flags

| Flag | Meaning |
|------|---------|
| `CQE_F_BUFFER` | Upper 16 bits of flags = buffer ID (provided buffers) |
| `CQE_F_MORE` | More CQEs coming from this SQE (multishot) |
| `CQE_F_SOCK_NONEMPTY` | Socket has more data (hint for recv) |
| `CQE_F_NOTIF` | Zero-copy send notification — buffer is now safe to reuse |
| `CQE_F_BUF_MORE` | Buffer partially consumed (incremental pbuf ring) |
| `CQE_F_SKIP` | Skip this CQE (mixed CQE gap filler) |
| `CQE_F_32` | This is a 32-byte CQE (mixed CQE mode) |

## Overflow

When the CQ ring is full:
1. Kernel tries to allocate overflow entries on the heap
2. Sets `IORING_SQ_CQ_OVERFLOW` flag in SQ ring
3. On next `io_uring_enter()`, overflowed CQEs are flushed back to CQ ring
4. `IORING_FEAT_NODROP` (always set since 5.5): kernel won't silently drop CQEs

Overflow is expensive (heap allocation, locking). Size your CQ ring to avoid it. See [cq-overflow.md](../patterns/cq-overflow.md).

## CQE_SKIP_SUCCESS

`IOSQE_CQE_SKIP_SUCCESS` on the SQE suppresses CQE generation on success. Only failures post CQEs. Reduces CQ pressure for fire-and-forget operations:

```c
sqe->flags |= IOSQE_CQE_SKIP_SUCCESS;
// If res >= 0: no CQE posted
// If res < 0: CQE posted with error
```

Use for: intermediate linked ops where you only care about the final result, NOP barriers, MSG_RING notifications.

## DEFER_TASKRUN Interaction

With `IORING_SETUP_DEFER_TASKRUN`, CQEs are not posted until the application enters the kernel via `io_uring_enter()`. This means:
- Polling CQ tail without entering the kernel shows stale data
- Must call `io_uring_enter(IORING_ENTER_GETEVENTS)` to get completions
- Maximum batching — all pending completions delivered at once

This is the highest-performance mode but requires the application to be designed around explicit completion polling.
