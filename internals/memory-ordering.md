# io_uring Memory Model

The shared memory contract between kernel and userspace. Get this wrong and you corrupt the rings.

## Ring Memory Layout

Both SQ and CQ are memory-mapped regions shared between kernel and userspace. The kernel writes CQEs; userspace writes SQEs. Coordination happens through head/tail counters.

```
Submission Queue:                    Completion Queue:
┌──────────────────────┐             ┌──────────────────────┐
│ head (kernel reads)  │ ◄── atomic  │ head (user advances) │ ◄── atomic
│ tail (user advances) │ ◄── atomic  │ tail (kernel writes)  │ ◄── atomic
│ mask                 │             │ mask                 │
│ flags                │             │ flags                │
│ array[mask+1]        │             │ cqes[mask+1]         │
└──────────────────────┘             └──────────────────────┘
```

Userspace owns: SQ tail, CQ head.
Kernel owns: SQ head, CQ tail.

## Ordering Requirements

### Userspace → Kernel (Submission)

1. Write SQE data to `sqes[idx]`
2. **Write barrier** (`smp_store_release` / `atomic_store_release`)
3. Update SQ tail

The barrier ensures the kernel sees complete SQE data before it sees the new tail. Without it, the kernel might read a partially-written SQE.

```c
// liburing: io_uring_submit()
io_uring_smp_store_release(sq->ktail, sq->sqe_tail);
```

### Kernel → Userspace (Completion)

1. Kernel writes CQE data to `cqes[idx]`
2. **Write barrier** (kernel-side)
3. Kernel updates CQ tail

Userspace must use a **read barrier** (`smp_load_acquire`) when reading the CQ tail:

```c
// liburing: io_uring_peek_cqe()
unsigned tail = io_uring_smp_load_acquire(cq->ktail);
```

### After Processing CQE

1. Process CQE data
2. **Write barrier**
3. Advance CQ head

```c
// liburing: io_uring_cq_advance()
io_uring_smp_store_release(cq->khead, *cq->khead + nr);
```

## Platform Barrier Implementations

liburing defines barriers per architecture:

### x86/x86_64

```c
#define io_uring_smp_store_release(p, v) \
    __atomic_store_n(p, v, __ATOMIC_RELEASE)
#define io_uring_smp_load_acquire(p) \
    __atomic_load_n(p, __ATOMIC_ACQUIRE)
```

On x86, `RELEASE` stores and `ACQUIRE` loads compile to plain loads/stores — the hardware provides TSO (Total Store Order). The compiler barriers prevent reordering.

### ARM/aarch64

Actual barrier instructions emitted. ARM has a relaxed memory model — without barriers, reordering is real.

### RISC-V

Uses fence instructions. Similar to ARM.

## The SQ Indirection Array

The SQ has an extra level of indirection: `sq.array[i]` maps to `sqes[sq.array[i]]`. This allows non-contiguous SQE consumption.

```
SQ:   array[0]=3, array[1]=1, array[2]=0, ...
SQEs: [sqe_0, sqe_1, sqe_2, sqe_3, ...]
      kernel reads: sqes[3], sqes[1], sqes[0]
```

In practice, most code fills SQEs sequentially and `array[i] = i`. But the indirection exists.

## SQPOLL Memory Considerations

With `IORING_SETUP_SQPOLL`, a kernel thread polls the SQ. No `io_uring_enter()` needed for submission.

The SQ `flags` field gains importance:
- Kernel sets `IORING_SQ_NEED_WAKEUP` when the SQ thread goes idle
- Userspace must check this flag and call `io_uring_enter(IORING_ENTER_SQ_WAKEUP)` if set
- This check must use `smp_load_acquire` — the flag is written by another thread (kernel SQ thread)

```c
if (io_uring_smp_load_acquire(sq->kflags) & IORING_SQ_NEED_WAKEUP) {
    io_uring_enter(ring_fd, 0, 0, IORING_ENTER_SQ_WAKEUP);
}
```

## CQE Ordering vs. SQE Ordering

**CQEs do not arrive in SQE submission order.** Operations complete as the kernel finishes them. If you need ordering, use:

- **IOSQE_IO_LINK**: Chain operations. Next starts when previous completes.
- **IOSQE_IO_DRAIN**: Wait for all prior submissions to complete.

Neither imposes memory ordering — they impose execution ordering.

## Common Mistakes

1. **Missing store-release on SQ tail**: Kernel reads stale/partial SQE → undefined behavior
2. **Missing load-acquire on CQ tail**: Userspace reads partial CQE → garbage data
3. **Forgetting CQ head advance**: CQ fills up → kernel can't post completions → deadlock
4. **Assuming CQE order**: First submitted ≠ first completed
5. **Non-atomic counter access**: Head/tail are `unsigned int` but must be accessed atomically
6. **Ignoring SQ_NEED_WAKEUP in SQPOLL**: SQ thread sleeps, submissions stop

## The io_uring_enter() Fence

`io_uring_enter()` itself acts as a full memory fence. After it returns, all submitted SQEs are visible to the kernel. But you still need the SQ tail store-release — the kernel may read the tail before processing the syscall if SQPOLL is active.
