# io_uring Request Lifecycle

From SQE submission to CQE delivery. Every step.

## Phase 1: SQE Preparation (Userspace)

Application fills an `io_uring_sqe` in the shared SQ ring:

```
1. Get next SQ slot: tail = *sq->ktail (load-acquire on non-x86)
2. Write SQE at sqes[tail & sq_mask]
3. Store-release: *sq->ktail = tail + 1
```

With `NO_SQARRAY` (6.7), the SQ index array is eliminated — SQEs are read directly from indices 0..N. With `SQ_REWIND` (6.19), head/tail are ignored entirely; kernel reads from index 0 each submission.

## Phase 2: Submission

Two paths:

### Syscall Path (default)
```
io_uring_enter(ring_fd, to_submit, min_complete, flags)
  → sys_io_uring_enter()
    → io_submit_sqes()
```

Each SQE is consumed: opcode decoded, request struct (`io_kiocb`) allocated from a slab cache. The SQ head advances as entries are consumed.

### SQPOLL Path
The kernel `sq_thread` polls `*sq->ktail` in a loop. No syscall needed unless the thread goes idle (`IORING_SQ_NEED_WAKEUP` set in `sq->flags`). Application checks this flag and calls `io_uring_enter(IORING_ENTER_SQ_WAKEUP)` to wake it.

## Phase 3: Request Processing

Each `io_kiocb` enters the execution pipeline:

```
io_issue_sqe(req)
  ├─ Inline completion (fast path)
  │   Operation completes immediately (e.g., cached read, buffer available)
  │   → io_req_complete_post()
  │
  ├─ Poll-armed (medium path)
  │   -EAGAIN → arm internal poll (io_arm_poll_handler)
  │   When poll triggers → retry the operation
  │
  └─ io-wq offload (slow path)
      Operation can't complete inline and can't be polled
      → io_wq_enqueue() dispatches to worker thread pool
      Bounded workers: file I/O, blocking ops
      Unbounded workers: network ops without fixed resource limits
```

### IOPOLL Path
With `IORING_SETUP_IOPOLL`, completions aren't interrupt-driven. Instead:
- `io_uring_enter(IORING_GETEVENTS)` calls block device `->poll()` directly
- Hybrid IOPOLL (6.13): initial sleep delay, then poll — best of both worlds

## Phase 4: Completion

When work finishes:

```
io_req_complete_post(req, res)
  1. Write CQE:
     cqe = &cq->cqes[cq_tail & cq_mask]
     cqe->user_data = req->user_data
     cqe->res = result
     cqe->flags = flags
  2. Store-release: *cq->ktail = cq_tail + 1
  3. If eventfd registered → eventfd_signal()
  4. Wake up any waiters (io_cqring_ev_posted)
```

### Task Work Delivery
With `COOP_TASKRUN`, completions are delivered as task work — no IPI. The submitting task processes them on its next kernel transition (syscall, signal, etc.).

With `DEFER_TASKRUN`, task work is deferred until the next `io_uring_enter(GETEVENTS)` call. Maximum userspace CPU time, but requires `SINGLE_ISSUER`.

### TASKRUN_FLAG
If `COOP_TASKRUN` + `TASKRUN_FLAG` are set, the kernel sets `IORING_SQ_TASKRUN` in `sq->flags` when task work is pending. Application polls this flag to decide when to enter the kernel.

## Phase 5: CQE Consumption (Userspace)

```
1. Load-acquire: tail = *cq->ktail
2. Compare with local head
3. Read CQE at cqes[head & cq_mask]
4. Process result
5. Advance head: *cq->khead = head + 1
```

## CQ Overflow

If CQ is full when kernel tries to post a CQE:
1. CQE goes to an internal overflow list
2. `IORING_SQ_CQ_OVERFLOW` set in `sq->flags`
3. On next `io_uring_enter()`, overflow list is flushed to CQ ring
4. With `IORING_FEAT_NODROP`: submissions block until CQ has space (no silent drops)

## Multishot CQEs

Multishot operations (accept, recv, poll) produce multiple CQEs from one SQE:
- Each CQE has `IORING_CQE_F_MORE` set (except the last)
- Final CQE: `F_MORE` cleared, usually carries an error or termination signal
- The original SQE slot is reused internally — no new submission needed

## Cancellation

```
IORING_OP_ASYNC_CANCEL
  ├─ Match by user_data (default)
  ├─ Match by fd (IORING_ASYNC_CANCEL_FD)
  ├─ Match by opcode (IORING_ASYNC_CANCEL_OP)
  └─ Match all (IORING_ASYNC_CANCEL_ALL)
```

Cancelled request gets CQE with `-ECANCELED`. The cancel op itself gets a CQE with 0 (success) or `-ENOENT` (not found).

## Linked Request Chains

Linked SQEs form dependency chains:
- **Soft link** (`IOSQE_IO_LINK`): next SQE starts when current completes successfully. On failure, rest of chain gets `-ECANCELED`.
- **Hard link** (`IOSQE_IO_HARDLINK`): next SQE starts regardless of current result.
- **Link timeout**: `IORING_OP_LINK_TIMEOUT` linked after another op — cancels the previous if it doesn't complete in time.

## Request Flags Summary

| SQE Flag | Effect on Lifecycle |
|----------|-------------------|
| `IOSQE_FIXED_FILE` | Use pre-registered fd, skip fdget/fdput |
| `IOSQE_IO_DRAIN` | Wait for all prior requests to complete first |
| `IOSQE_ASYNC` | Skip inline attempt, go straight to io-wq |
| `IOSQE_BUFFER_SELECT` | Pick buffer from provided buffer pool at completion time |
| `IOSQE_CQE_SKIP_SUCCESS` | Don't post CQE on success — only on failure |

## Performance Characteristics

| Phase | Overhead |
|-------|----------|
| SQE write | ~10ns (shared memory write) |
| Syscall (io_uring_enter) | ~200-400ns (amortized across batch) |
| SQPOLL submission | ~0ns userspace (kernel polls) |
| Inline completion | ~100-200ns |
| io-wq offload | ~1-5µs (thread wakeup + context switch) |
| CQE read | ~10ns (shared memory read) |

Batch submission is the key optimization. One syscall submitting 32 SQEs costs roughly the same as submitting 1.
