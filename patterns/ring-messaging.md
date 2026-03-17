# Ring-to-Ring Messaging

Cross-ring communication patterns. How threads and processes coordinate through io_uring without shared state.

## IORING_OP_MSG_RING

Send a message from one ring to another. The target ring gets a CQE with your data. No shared memory, no locks, no eventfd.

### Data Messages

```c
// Thread A: send data to Thread B's ring
sqe = io_uring_get_sqe(ring_a);
io_uring_prep_msg_ring(sqe, ring_b_fd, result_value, user_data, 0);
io_uring_submit(ring_a);
```

Thread B sees a CQE with:
- `cqe->res` = `result_value` (the "message")
- `cqe->user_data` = `user_data` (your tag)

No actual I/O happens. The kernel just posts a CQE to the target ring. ~100ns.

### File Descriptor Passing

```c
sqe = io_uring_get_sqe(ring_a);
io_uring_prep_msg_ring_fd(sqe, ring_b_fd, src_fd, dst_slot, user_data, 0);
```

Sends a file descriptor from ring A's fixed file table to ring B's. The accept-on-one-ring, process-on-another pattern.

### Flags

- `IORING_MSG_DATA`: Pass a value (default)
- `IORING_MSG_SEND_FD`: Pass a file descriptor
- `IORING_MSG_RING_CQE_SKIP`: Don't post CQE on target ring (fire-and-forget to target)
- `IORING_MSG_RING_FLAGS_PASS`: Forward `file_index` to `cqe->flags` on target

## IORING_REGISTER_SEND_MSG_RING (6.10)

The registration variant. Send a message to a ring **without owning a ring yourself**.

```c
struct io_uring_sqe sqe = {
    .opcode = IORING_OP_MSG_RING,
    .fd = target_ring_fd,
    .len = result_value,
    .off = user_data,
};
io_uring_register(ring_fd, IORING_REGISTER_SEND_MSG_RING, &sqe, 1);
```

Use case: worker threads that don't have their own ring can signal the event loop thread. No eventfd overhead.

## Patterns

### Accept → Dispatch (Connection Distribution)

One ring accepts connections, distributes to worker rings:

```
Acceptor Ring (Thread 0)
  ├─ multishot accept → new fd
  ├─ MSG_RING(worker_1_fd, IORING_MSG_SEND_FD, new_fd, slot)
  ├─ MSG_RING(worker_2_fd, IORING_MSG_SEND_FD, new_fd, slot)
  └─ Round-robin or least-loaded

Worker Ring (Thread N)
  ├─ Receives CQE with fd in fixed slot
  └─ Starts multishot recv on that fd
```

### Work Stealing

Ring A is overloaded, ring B is idle:

```c
// Ring A: hand off pending work
sqe = io_uring_get_sqe(ring_a);
io_uring_prep_msg_ring(sqe, ring_b_fd, WORK_ITEM_ID, task_ptr, 0);
```

Ring B picks up the CQE, uses `user_data` as a pointer to the work item. Zero-copy task migration between threads.

### Shutdown Coordination

Broadcast shutdown to all worker rings:

```c
for (int i = 0; i < num_workers; i++) {
    sqe = io_uring_get_sqe(coordinator_ring);
    io_uring_prep_msg_ring(sqe, worker_fds[i], SHUTDOWN_SIGNAL, 0, 0);
}
io_uring_submit(coordinator_ring);
```

Workers check `cqe->res == SHUTDOWN_SIGNAL` in their event loop. Clean, no signals, no shared flags.

### Completion Forwarding

Operation completes on ring A, result needed on ring B:

```c
// Ring A completion handler:
void on_complete(struct io_uring_cqe *cqe) {
    // Forward result to ring B
    sqe = io_uring_get_sqe(ring_a);
    io_uring_prep_msg_ring(sqe, ring_b_fd, cqe->res, cqe->user_data, 0);
    io_uring_submit(ring_a);
}
```

## vs Alternatives

| Mechanism | Latency | Syscalls | Kernel Support |
|-----------|---------|----------|---------------|
| `MSG_RING` | ~100ns | 0 (batched) | 5.18+ |
| `eventfd` | ~200-500ns | write+read | Always |
| `pipe` | ~500ns-1µs | write+read | Always |
| `futex` | ~200ns | wake+wait | Always |
| Shared memory + poll | ~50ns | 0 | Always |

MSG_RING wins when you're already in the io_uring submission path. The message piggybacks on the next `io_uring_enter()` — no extra syscall.

Shared memory + atomic poll is faster if you need sub-100ns, but you lose the structured CQE delivery and the kernel handles cross-thread wakeups for you with MSG_RING.

## Gotchas

1. **Target ring must exist**: Sending to a closed ring fd → `-EBADFD`
2. **CQ overflow**: If target ring's CQ is full, the MSG_RING CQE goes to overflow list — same rules as any other CQE
3. **No ordering guarantees**: MSG_RING CQEs interleave with I/O CQEs on the target ring
4. **Fixed fd required for SEND_FD**: Source fd must be in sender's fixed file table
5. **Cross-process**: Works if you pass the ring fd via SCM_RIGHTS. Same as any fd passing.
