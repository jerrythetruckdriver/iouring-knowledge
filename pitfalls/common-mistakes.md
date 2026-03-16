# Common Mistakes

## 1. Not Checking for CQ Overflow

If CQ fills up, completions are dropped (unless `IORING_FEAT_NODROP`). Check `IORING_SQ_CQ_OVERFLOW` flag. With multishot ops, this is easy to hit.

**Fix**: Size CQ at 4-8x SQ entries. Process CQEs promptly.

## 2. Buffer Lifetime with Completion-Based I/O

Unlike epoll (readiness-based), io_uring takes ownership of your buffer until completion. Accessing the buffer between submit and completion is undefined behavior.

```c
// WRONG
io_uring_prep_read(sqe, fd, buf, len, 0);
io_uring_submit(ring);
memset(buf, 0, len);  // BUG: kernel might be writing to buf
```

**Fix**: Don't touch buffers between submit and CQE. Use ownership-transfer patterns.

## 3. Forgetting IORING_CQE_F_MORE

Multishot operations produce CQEs with `IORING_CQE_F_MORE` set. When it's cleared, the multishot is done. If you don't check, you'll miss the termination and never resubmit.

```c
// Must check
if (!(cqe->flags & IORING_CQE_F_MORE)) {
    resubmit_multishot(ring);
}
```

## 4. Not Handling Partial Reads/Writes

`cqe->res` can be less than requested. Same as regular read/write — short reads happen.

```c
// WRONG: assuming full read
io_uring_prep_read(sqe, fd, buf, 4096, 0);
// CQE: res = 1024 (only read 1024 bytes)
```

**Fix**: Handle short reads. Submit another read for the remainder.

## 5. SQPOLL Without Checking NEED_WAKEUP

SQ poll thread sleeps after idle timeout. If you submit without checking, the SQE sits unprocessed.

```c
// Must check after writing SQE
if (*ring->sq.kflags & IORING_SQ_NEED_WAKEUP) {
    io_uring_enter(fd, 0, 0, IORING_ENTER_SQ_WAKEUP);
}
```

liburing handles this automatically in `io_uring_submit()`.

## 6. Not Using SINGLE_ISSUER When Applicable

If only one thread submits to a ring (which is almost always the correct design), set `SINGLE_ISSUER`. It enables significant internal optimizations. Required for `DEFER_TASKRUN`.

## 7. CQ Ring Too Small for Multishot

Default CQ is 2x SQ. With multishot accept + multishot recv, one SQE can generate thousands of CQEs. Size your CQ ring explicitly:

```c
params.flags |= IORING_SETUP_CQSIZE;
params.cq_entries = 16384; // explicit size
```

## 8. Ignoring IORING_CQE_F_NOTIF for Zero-Copy Send

Zero-copy send produces TWO CQEs: the completion and a notification. Don't free the buffer on the first CQE — wait for the notification (`IORING_CQE_F_NOTIF`).

## 9. Using DRAIN When You Mean LINK

`IOSQE_IO_DRAIN` serializes against ALL previous SQEs. `IOSQE_IO_LINK` only orders against the next SQE. Drain is a sledgehammer.

## 10. Not Probing Opcode Support

```c
// Right way
struct io_uring_probe *p = io_uring_get_probe_ring(&ring);
if (!io_uring_opcode_supported(p, IORING_OP_SEND_ZC)) {
    // Fall back gracefully
}
```

Never assume opcode support based on kernel version. Always probe.

## 11. Provided Buffer Ring Exhaustion

When all buffers are consumed and none returned, multishot recv terminates. Your server silently stops receiving.

**Fix**: Monitor buffer availability. Return buffers immediately after processing.

## 12. Fixed File Table Full

With `IORING_FILE_INDEX_ALLOC`, if the table is full, accept returns `-ENFILE`. 

**Fix**: Register enough file slots. Use `IORING_REGISTER_FILE_ALLOC_RANGE` to manage ranges.
