# Error Recovery Patterns

## CQE Error Convention

`cqe->res` is negative errno on failure, positive byte count (or 0) on success. Always check before consuming.

```c
if (cqe->res < 0) {
    int err = -cqe->res;
    // handle error
}
```

## Retry-Safe vs Non-Retry-Safe Operations

**Safe to retry:**
- READ, WRITE (idempotent if same offset)
- RECV, SEND (stream sockets — protocol handles ordering)
- ACCEPT (stateless)
- POLL_ADD (stateless)
- STATX, FADVISE, MADVISE (read-only)

**NOT safe to blindly retry:**
- WRITE without offset (append mode — may double-write)
- RENAME, UNLINK, MKDIR (may have already completed)
- SEND_ZC (notification CQE may still arrive after error CQE)
- Linked chains (partial completion — see below)

## Partial Failure in Linked Chains

When a linked chain fails mid-way:

```
SQE_A (LINK) → SQE_B (LINK) → SQE_C
```

If SQE_B fails:
- SQE_A: already completed successfully (CQE delivered)
- SQE_B: CQE with negative res
- SQE_C: CQE with `-ECANCELED` (soft link) or executed anyway (hard link)

**Recovery pattern:**
1. Track chain state via user_data (encode chain_id + step)
2. On failure CQE, drain remaining CQEs for that chain
3. Compensate for already-completed steps (e.g., if write succeeded but fsync failed, the write is durable but not synced)

```c
// Encode chain info in user_data
#define MAKE_UD(chain, step) (((uint64_t)(chain) << 32) | (step))
#define CHAIN_ID(ud) ((ud) >> 32)
#define STEP(ud) ((ud) & 0xFFFFFFFF)
```

## Short Read / Short Write

`cqe->res` may be less than requested `len`. This is normal, not an error.

**Common causes:**
- EOF (file shorter than expected)
- Socket buffer limits
- Signal interruption (though io_uring auto-restarts most)
- Pipe buffer full

**Pattern:** Resubmit with adjusted offset and length:

```c
if (cqe->res > 0 && cqe->res < expected_len) {
    // Short read — resubmit for remainder
    new_sqe->off = original_off + cqe->res;
    new_sqe->len = expected_len - cqe->res;
    new_sqe->addr = original_addr + cqe->res;
}
```

QEMU does exactly this in `luring_resubmit_short_read()`.

## EAGAIN Handling

io_uring normally handles EAGAIN internally (re-arms poll, retries). But you may see it in edge cases:

- SQPOLL ring when SQ is full
- File I/O that can't be completed inline

If you get EAGAIN, resubmit. Don't spin — add a brief delay or wait for a CQ event.

## CQ Overflow Recovery

When CQ is full, kernel allocates overflow entries on heap. Detect via:

```c
unsigned overflow = *ring->cq.koverflow;
if (overflow > 0) {
    // CQEs were delayed — drain CQ aggressively
}
```

Recovery: drain all available CQEs, resize ring if chronic (`REGISTER_RESIZE_RINGS`, 6.13), or submit fewer SQEs per batch.

See [cq-overflow.md](cq-overflow.md) for complete strategies.

## Multishot Termination

Multishot operations (accept, recv, poll) terminate when CQE lacks `CQE_F_MORE`:

```c
if (!(cqe->flags & IORING_CQE_F_MORE)) {
    // Multishot terminated — must resubmit
    if (cqe->res < 0) {
        // Error caused termination
    } else {
        // CQ overflow or resource exhaustion
    }
    resubmit_multishot(ring, original_sqe_params);
}
```

**Always** check `CQE_F_MORE`. Multishot can terminate silently on CQ pressure.

## Cancel + Cleanup

Graceful shutdown pattern:

```c
// 1. Cancel everything
sqe->opcode = IORING_OP_ASYNC_CANCEL;
sqe->cancel_flags = IORING_ASYNC_CANCEL_ANY;

// 2. Drain all CQEs (cancelled ops produce -ECANCELED CQEs)
while (pending_ops > 0) {
    io_uring_wait_cqe(&ring, &cqe);
    pending_ops--;
    io_uring_cqe_seen(&ring, cqe);
}

// 3. Now safe to destroy ring
io_uring_queue_exit(&ring);
```

Never destroy a ring with in-flight operations. The kernel will complete them, but you won't see the CQEs, and registered resources may leak.

## Error Logging

Minimum useful error context:

```c
fprintf(stderr, "io_uring op=%d fd=%d res=%d (%s) user_data=0x%llx\n",
        op_type, fd, cqe->res, strerror(-cqe->res),
        (unsigned long long)cqe->user_data);
```

Include the opcode, fd, and user_data. Without these, io_uring errors are nearly impossible to diagnose.
