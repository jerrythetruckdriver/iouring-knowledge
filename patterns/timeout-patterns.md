# Timeout Patterns

Comprehensive reference for io_uring timeout operations.

## Timeout Opcodes

| Op | Opcode | Since | Description |
|----|--------|-------|-------------|
| `IORING_OP_TIMEOUT` | 11 | 5.4 | Wait for time or N completions |
| `IORING_OP_TIMEOUT_REMOVE` | 12 | 5.5 | Cancel/update a pending timeout |
| `IORING_OP_LINK_TIMEOUT` | 15 | 5.5 | Timeout for a linked operation |

## Timeout Flags

```c
#define IORING_TIMEOUT_ABS           (1U << 0)  // Absolute timestamp
#define IORING_TIMEOUT_UPDATE        (1U << 1)  // Update existing timeout
#define IORING_TIMEOUT_BOOTTIME      (1U << 2)  // CLOCK_BOOTTIME
#define IORING_TIMEOUT_REALTIME      (1U << 3)  // CLOCK_REALTIME
#define IORING_LINK_TIMEOUT_UPDATE   (1U << 4)  // Update linked timeout
#define IORING_TIMEOUT_ETIME_SUCCESS (1U << 5)  // Return 0 on timeout (not -ETIME)
#define IORING_TIMEOUT_MULTISHOT     (1U << 6)  // Repeating timeout
#define IORING_TIMEOUT_IMMEDIATE_ARG (1U << 7)  // addr = nanoseconds (not timespec*)
```

Default clock: `CLOCK_MONOTONIC`. Override with `BOOTTIME` or `REALTIME` flags. Or register a custom clock with `IORING_REGISTER_CLOCK` (6.13).

## Basic Timeout

```c
struct __kernel_timespec ts = { .tv_sec = 5 };
io_uring_prep_timeout(sqe, &ts, 0, 0);
// CQE: res = -ETIME on expiry
```

Second argument is completion count — timeout fires after N completions OR time expiry, whichever first:

```c
io_uring_prep_timeout(sqe, &ts, 10, 0);
// Fires after 10 CQEs or 5 seconds
```

## Absolute Timeout

```c
struct __kernel_timespec deadline;
clock_gettime(CLOCK_MONOTONIC, &deadline);
deadline.tv_sec += 30;  // 30s from now

io_uring_prep_timeout(sqe, &deadline, 0, IORING_TIMEOUT_ABS);
```

Use absolute timeouts when you need coordination with wall clock or when computing from a known deadline. Avoids drift from submission delay.

## Immediate Argument (No Timespec Allocation)

```c
// 6.x+ with IORING_TIMEOUT_IMMEDIATE_ARG
sqe->addr = 5000000000ULL;  // 5 seconds in nanoseconds
sqe->timeout_flags = IORING_TIMEOUT_IMMEDIATE_ARG;
```

Avoids allocating and pointing to a `__kernel_timespec`. Useful in hot paths where you don't want the struct on the stack outliving submission.

## Multishot Timeout

```c
struct __kernel_timespec interval = { .tv_sec = 1 };
io_uring_prep_timeout(sqe, &interval, 0, IORING_TIMEOUT_MULTISHOT);
// CQE every 1 second with CQE_F_MORE set
// Last CQE (after cancel) won't have CQE_F_MORE
```

Repeating timer. Generates one CQE per interval until canceled. Ideal for periodic tasks, heartbeats, stats collection. One SQE, unlimited CQEs.

## Link Timeout

Enforces a deadline on a preceding linked operation:

```c
// SQE 1: the actual operation
io_uring_prep_recv(sqe1, sock_fd, buf, len, 0);
sqe1->flags |= IOSQE_IO_LINK;

// SQE 2: link timeout (kills sqe1 if it takes >5s)
struct __kernel_timespec ts = { .tv_sec = 5 };
io_uring_prep_link_timeout(sqe2, &ts, 0);
```

**Completion semantics:**
- Operation completes first → link timeout gets `-ECANCELED`
- Timeout fires first → operation gets `-ECANCELED`, timeout gets `-ETIME`

This is the pattern for per-request timeouts. No need for external timer management.

## Timeout Update

Change a pending timeout's expiry without canceling and resubmitting:

```c
struct __kernel_timespec new_ts = { .tv_sec = 10 };
io_uring_prep_timeout_update(sqe, &new_ts, user_data_of_original, 0);
```

`user_data_of_original` identifies which timeout to update. Useful for connection idle timeouts — reset on activity without churn.

## Link Timeout Update

```c
io_uring_prep_link_timeout_update(sqe, &new_ts, user_data_of_original,
                                   IORING_LINK_TIMEOUT_UPDATE);
```

## Registered Clock (6.13)

```c
struct io_uring_clock_register cr = { .clockid = CLOCK_MONOTONIC_RAW };
io_uring_register_clock(ring, &cr);
```

After registration, all timeouts on this ring use the registered clock by default. Override per-timeout with `BOOTTIME`/`REALTIME` flags.

Use `CLOCK_MONOTONIC_RAW` for benchmarking (no NTP adjustment). Use `CLOCK_BOOTTIME` when timeouts should account for system suspend.

## Timeout + Registered Wait

Combine with `io_uring_reg_wait` for batched waiting with timeout:

```c
struct io_uring_reg_wait rw = {
    .ts = { .tv_sec = 0, .tv_nsec = 100000000 },  // 100ms max wait
    .min_wait_usec = 10000,  // batch for at least 10ms
    .flags = IORING_REG_WAIT_TS,
};
```

This isn't a timeout SQE — it's a wait parameter. But it controls how long `io_uring_enter` blocks, with a minimum batch window. The combo: registered wait for submission batching, timeout SQEs for per-request deadlines.

## Pattern: Connection Idle Timeout

```c
// On new connection: arm timeout
io_uring_prep_timeout(sqe, &idle_ts, 0, 0);
sqe->user_data = encode(conn_id, TIMEOUT_TAG);

// On activity: update timeout
io_uring_prep_timeout_update(sqe, &idle_ts, encode(conn_id, TIMEOUT_TAG), 0);

// On timeout CQE: close connection
```

## Pattern: Request Deadline Chain

```c
// Linked chain: connect → send → recv, with 30s overall deadline
io_uring_prep_connect(sqe1, ...); sqe1->flags |= IOSQE_IO_LINK;
io_uring_prep_send(sqe2, ...);    sqe2->flags |= IOSQE_IO_LINK;
io_uring_prep_recv(sqe3, ...);    sqe3->flags |= IOSQE_IO_LINK;
io_uring_prep_link_timeout(sqe4, &deadline, 0);
```

Link timeout applies to the entire chain. If any op in the chain is still pending when timeout fires, the pending op and timeout both get canceled.

## Pattern: Periodic Stats with Multishot

```c
io_uring_prep_timeout(sqe, &one_second, 0, IORING_TIMEOUT_MULTISHOT);
sqe->user_data = STATS_TIMER;

// In completion loop:
if (cqe->user_data == STATS_TIMER) {
    print_stats();
    // No resubmission needed — CQE_F_MORE means another is coming
}
```

## ETIME_SUCCESS Flag

By default, timeout CQEs return `res = -ETIME`. With `IORING_TIMEOUT_ETIME_SUCCESS`:

```c
io_uring_prep_timeout(sqe, &ts, 0, IORING_TIMEOUT_ETIME_SUCCESS);
// CQE: res = 0 on timeout (not -ETIME)
```

Simplifies completion handling when you treat timeout as a normal event, not an error.
