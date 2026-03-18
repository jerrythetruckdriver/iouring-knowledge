# Cancel Patterns

## IORING_OP_ASYNC_CANCEL (opcode 14)

Cancels in-flight requests. Available since 5.5.

### SQE Layout

```c
sqe->opcode  = IORING_OP_ASYNC_CANCEL;
sqe->addr    = target_user_data;  // default: match by user_data
sqe->fd      = target_fd;         // when CANCEL_FD set
sqe->cancel_flags = flags;
```

### Cancel Flags

| Flag | Value | Since | Behavior |
|------|-------|-------|----------|
| `IORING_ASYNC_CANCEL_ALL` | 1<<0 | 5.19 | Cancel **all** matching requests, not just first |
| `IORING_ASYNC_CANCEL_FD` | 1<<1 | 5.19 | Key on `fd` instead of `user_data` |
| `IORING_ASYNC_CANCEL_ANY` | 1<<2 | 6.0 | Match **any** request (ignores addr/fd) |
| `IORING_ASYNC_CANCEL_FD_FIXED` | 1<<3 | 6.0 | `fd` is a fixed descriptor index |
| `IORING_ASYNC_CANCEL_USERDATA` | 1<<4 | 6.1 | Explicit: match on `user_data` |
| `IORING_ASYNC_CANCEL_OP` | 1<<5 | 6.1 | Match on opcode (set in `sqe->addr`) |

### CQE Result

| Value | Meaning |
|-------|---------|
| 0 | Found and canceled |
| -ENOENT | No matching request found |
| -EALREADY | Request found but already completing |
| N (positive) | With `CANCEL_ALL`: number of requests canceled |

### Synchronous Cancel

`IORING_REGISTER_SYNC_CANCEL` (opcode 24) blocks until cancel completes or times out.

```c
struct io_uring_sync_cancel_reg {
    __u64   addr;       // user_data to match
    __s32   fd;         // fd to match (with FD flag)
    __u32   flags;      // same ASYNC_CANCEL_* flags
    struct __kernel_timespec timeout;
    __u8    opcode;     // opcode to match (with OP flag)
    __u8    pad[7];
    __u64   pad2[3];
};
```

Useful at shutdown: blocks the calling thread until the target request is actually done.

### Patterns

**Cancel by user_data (default):**
```c
io_uring_prep_cancel64(sqe, target_user_data, 0);
```

**Cancel all requests on a fd:**
```c
sqe->cancel_flags = IORING_ASYNC_CANCEL_FD | IORING_ASYNC_CANCEL_ALL;
sqe->fd = socket_fd;
```

**Cancel all recv ops:**
```c
sqe->cancel_flags = IORING_ASYNC_CANCEL_OP | IORING_ASYNC_CANCEL_ALL;
sqe->addr = IORING_OP_RECV;
```

**Nuclear option — cancel everything:**
```c
sqe->cancel_flags = IORING_ASYNC_CANCEL_ANY | IORING_ASYNC_CANCEL_ALL;
```

### What's Cancelable

- Poll-armed requests (recv, accept, connect waiting): always cancelable
- io-wq requests in progress: depends on operation. Disk I/O already submitted to the block layer? Not cancelable.
- Timeouts: cancelable via TIMEOUT_REMOVE (separate opcode)
- Linked chains: canceling the head cancels the chain

### Shutdown Pattern

```c
// 1. Cancel everything
sqe->cancel_flags = IORING_ASYNC_CANCEL_ANY | IORING_ASYNC_CANCEL_ALL;

// 2. Wait for all CQEs to drain
// Each canceled request still produces a CQE (with -ECANCELED)

// 3. Or use sync cancel with timeout for guaranteed cleanup
struct io_uring_sync_cancel_reg reg = {
    .flags = IORING_ASYNC_CANCEL_ANY | IORING_ASYNC_CANCEL_ALL,
    .timeout = { .tv_sec = 5 },
};
io_uring_register_sync_cancel(ring, &reg);
```

The evolution from "cancel by user_data only" to "cancel by fd/opcode/any" shows io_uring maturing into a real production-ready API. The early days were painful.
