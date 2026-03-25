# timerfd vs io_uring Timeouts

## Two Approaches to Async Timers

**timerfd:** Kernel creates a file descriptor that becomes readable when the timer fires. Bridge to io_uring via POLL_ADD.

**io_uring timeout:** Native IORING_OP_TIMEOUT. No fd, no kernel object, no poll overhead.

## Comparison

| Aspect | timerfd + POLL_ADD | IORING_OP_TIMEOUT |
|--------|-------------------|-------------------|
| Kernel objects | timerfd (anon inode, hrtimer) | None (inline in ring) |
| fd consumption | 1 per timer | 0 |
| Setup cost | timerfd_create + timerfd_settime + prep_poll | prep_timeout (one call) |
| Multishot | POLL_ADD multishot on timerfd | IORING_TIMEOUT_MULTISHOT (6.7) |
| Clock sources | CLOCK_MONOTONIC/REALTIME/BOOTTIME | Same + registered clock (6.13) |
| Cancellation | POLL_REMOVE or close fd | TIMEOUT_REMOVE or ASYNC_CANCEL |
| Update | timerfd_settime (syscall) | IORING_TIMEOUT_UPDATE (in-band) |
| Resolution | hrtimer (ns) | hrtimer (ns) |
| Batching | No (poll is per-fd) | Yes (min_wait_usec, registered wait) |
| Memory | ~400 bytes kernel + poll request | ~280 bytes (io_kiocb only) |

## When timerfd Still Wins

1. **Existing code** — library gives you a timerfd, bridge it with POLL_ADD rather than rewriting
2. **Cross-subsystem** — timer shared between io_uring and epoll event loops
3. **TFD_TIMER_ABSTIME + TFD_TIMER_CANCEL_ON_SET** — system clock change detection (io_uring timeout doesn't support this)
4. **Reading expiration count** — timerfd_read gives number of expirations since last read

## When io_uring Timeout Wins

1. **Everything else** — no fd overhead, no extra kernel objects, native cancellation/update
2. **Link timeout** — deadline for a chain of linked operations
3. **min_wait_usec** — batching completions with a timeout floor
4. **Registered wait** — pre-registered timeout parameters, zero-copy to kernel

## io_uring Timeout Patterns

```c
// One-shot relative timeout
struct __kernel_timespec ts = { .tv_sec = 5 };
io_uring_prep_timeout(sqe, &ts, 0, 0);

// Multishot periodic (6.7) — repeats until canceled
io_uring_prep_timeout(sqe, &ts, 0, IORING_TIMEOUT_MULTISHOT);

// Absolute timeout with registered clock (6.13)
io_uring_prep_timeout(sqe, &ts, 0,
    IORING_TIMEOUT_ABS | IORING_TIMEOUT_BOOTTIME);

// Link timeout — deadline for previous op
io_uring_prep_link_timeout(sqe, &ts, 0);

// Update existing timeout
io_uring_prep_timeout_update(sqe, &new_ts, old_user_data, 0);
```

## timerfd Bridge Pattern

```c
// When you MUST use timerfd (library gives you one)
int tfd = timerfd_create(CLOCK_MONOTONIC, TFD_NONBLOCK);
struct itimerspec its = { .it_value = { .tv_sec = 1 },
                          .it_interval = { .tv_sec = 1 } };
timerfd_settime(tfd, 0, &its, NULL);

// Multishot poll on timerfd
io_uring_prep_poll_multishot(sqe, tfd, POLLIN);
// Each CQE = timer fired. Read tfd to get expiration count if needed.
```

## Recommendation

Use io_uring native timeouts for everything new. Bridge timerfd with POLL_ADD only when you inherit one from external code.

## See Also

- [Timeout Patterns](timeout-patterns.md) — complete timeout reference
- [Multishot Timeout Rate Limiting](multishot-timeout-ratelimit.md) — periodic patterns
- [eventfd Integration](eventfd-integration.md) — similar bridging pattern
- [Poll Operations](poll-ops.md) — POLL_ADD for fd-based event sources
