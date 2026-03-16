# io_uring Error Handling Patterns

CQE `res` is your errno (negated). No exceptions, no error codes in a side channel. Just a signed int.

## CQE Result Convention

```
res > 0  → success, value is operation-specific (bytes transferred, fd, etc.)
res == 0 → success for void operations (close, fsync, connect)
           OR zero bytes transferred (check context)
res < 0  → -errno
```

## Universal Errors

These can come from any operation:

| errno | Meaning | Action |
|-------|---------|--------|
| `-ECANCELED` (-125) | Operation canceled | Expected during shutdown. Don't retry. |
| `-EINTR` (-4) | Signal interrupted | Resubmit. io_uring doesn't auto-retry. |
| `-EAGAIN` (-11) | Resource temporarily unavailable | Resubmit or back off. |
| `-EBADF` (-9) | Bad file descriptor | Bug. Your fd is wrong or closed. |
| `-EINVAL` (-22) | Invalid argument | Bug. Check SQE fields. |
| `-EOPNOTSUPP` (-95) | Operation not supported | Kernel too old or wrong fd type. |
| `-ENOMEM` (-12) | Out of memory | Back off, reduce in-flight ops. |

## Handling EINTR

Unlike syscalls, io_uring does NOT automatically restart on `EINTR`. You must resubmit:

```c
void handle_cqe(struct io_uring_cqe *cqe) {
    if (cqe->res == -EINTR) {
        // Resubmit the same operation
        resubmit(cqe->user_data);
        return;
    }
    // ... handle normally
}
```

TigerBeetle's approach — re-enqueue inside the completion handler:

```zig
.INTR => {
    completion.io.enqueue(completion);
    return;
},
```

## Submission Errors vs. Completion Errors

**Submission errors** (from `io_uring_enter()`):
- `-EAGAIN`: SQ full or CQ overcommitted → drain CQ, retry
- `-EBUSY`: Ring busy → retry
- `-ENOMEM`: Can't allocate request → reduce batch size

**Completion errors** (in CQE `res`):
- Operation-specific errno values
- Always delivered per-operation

These are different failure modes. Submission errors affect the batch; completion errors affect individual ops.

## Link Chain Errors

With `IOSQE_IO_LINK`, a failing SQE cancels the rest of the chain:

```
SQE_A (linked) → SQE_B (linked) → SQE_C
```

If SQE_A fails:
- CQE_A: res = -errno (the actual error)
- CQE_B: res = -ECANCELED
- CQE_C: res = -ECANCELED

With hard links (`IOSQE_IO_HARDLINK`), the chain continues even on failure.

Tracepoint: `io_uring:io_uring_fail_link` fires when a link breaks.

## CQ Overflow

If the CQ is full when the kernel tries to post a CQE:

1. Kernel stores it in an internal overflow list
2. Sets `IORING_SQ_CQ_OVERFLOW` flag in SQ flags
3. Next `io_uring_enter()` drains overflow into CQ

If you never drain:
- Overflow list grows, eating kernel memory
- With `IORING_SETUP_CQ_NODROP` (5.19+): submission blocks until CQ has space
- Without it: CQEs can be dropped silently in extreme cases

**Prevention**: Size CQ at 2× SQ entries. Drain CQ frequently.

## Short Reads/Writes

`res > 0` doesn't mean the full buffer was transferred:

```c
if (cqe->res > 0 && cqe->res < expected_bytes) {
    // Short read/write — resubmit with adjusted offset and buffer
    resubmit_partial(cqe->user_data, cqe->res);
}
```

For `IORING_OP_READ`/`WRITE` with regular files: short I/O means EOF or disk full.
For sockets: short I/O is normal. Always handle it.

## Multishot Error Handling

Multishot operations (`IORING_OP_ACCEPT` with `IOSQE_MULTISHOT`, etc.) produce multiple CQEs:

- `CQE_F_MORE` set → more CQEs coming, operation still active
- `CQE_F_MORE` not set → operation terminated, must resubmit if desired
- Error CQE → operation terminated regardless of `CQE_F_MORE`

```c
if (cqe->res < 0) {
    // Multishot terminated with error
    handle_error(cqe);
    // Resubmit multishot if you want it to continue
    resubmit_multishot();
} else if (!(cqe->flags & IORING_CQE_F_MORE)) {
    // Multishot ended (CQ overflow, etc.)
    resubmit_multishot();
}
```

## Timeout Handling

`IORING_OP_TIMEOUT` completion:
- `res = -ETIME`: Timer expired (success, really)
- `res = 0`: Timer completed because count was reached
- `res = -ECANCELED`: Timer was canceled

`IORING_OP_LINK_TIMEOUT` (linked to another op):
- `res = -ETIME`: Timeout fired, linked op was canceled
- `res = -ECANCELED`: Linked op completed before timeout
- `res = -EALREADY`: Linked op already in progress, too late to cancel

## Robust Error Handling Template

```c
void process_cqe(struct io_uring_cqe *cqe) {
    struct request *req = io_uring_cqe_get_data(cqe);

    if (cqe->res >= 0) {
        handle_success(req, cqe->res);
        return;
    }

    int err = -cqe->res;
    switch (err) {
    case EINTR:
    case EAGAIN:
        resubmit(req);  // Transient — retry
        break;
    case ECANCELED:
        cleanup(req);   // Expected during shutdown
        break;
    case ECONNRESET:
    case EPIPE:
    case ECONNREFUSED:
        close_connection(req);  // Peer gone
        break;
    case ENOMEM:
    case ENOSPC:
        backoff_and_retry(req); // Resource pressure
        break;
    default:
        log_error(req, err);    // Unexpected — log and decide
        break;
    }
}
```

## Kernel Version Gotchas

- Pre-5.5: Some ops return errors in different fields
- 5.18+: Mixed CQE sizes (32-byte extra data) — check `IORING_SETUP_CQE32`
- 6.0+: `IORING_OP_ASYNC_CANCEL` returns different errors for batch cancel
