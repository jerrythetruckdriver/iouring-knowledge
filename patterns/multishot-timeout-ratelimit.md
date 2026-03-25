# Multishot Timeout for Rate Limiting

Multishot timeouts (6.4) fire repeatedly at a fixed interval. Combined with other ops, they enable rate limiting, periodic tasks, and token bucket patterns — all without syscalls.

## Basic Multishot Timeout

```c
struct __kernel_timespec ts = { .tv_sec = 0, .tv_nsec = 100000000 }; // 100ms
io_uring_prep_timeout(sqe, &ts, 0, IORING_TIMEOUT_MULTISHOT);
sqe->user_data = RATE_LIMIT_TIMER;
```

This produces a CQE every 100ms with `CQE_F_MORE` set. Runs until cancelled.

## Token Bucket Pattern

```
multishot timeout (every T ms) → CQE → add N tokens to bucket
recv multishot → CQE → consume 1 token per request

Event loop:
  for each CQE:
    if user_data == TIMER:    tokens = min(tokens + refill_rate, max_tokens)
    if user_data == REQUEST:  if tokens > 0: process, tokens--
                              else: queue or drop
```

No syscalls for timing. The timer and I/O completions arrive on the same CQ.

## Rate-Limited Accept

Limit new connections per second:

```
multishot ACCEPT (listening socket)
multishot TIMEOUT (1 second interval)

accept_count = 0
max_per_second = 1000

on ACCEPT CQE:
  accept_count++
  if accept_count >= max_per_second:
    cancel multishot ACCEPT
    // stop accepting until next timer tick

on TIMEOUT CQE:
  accept_count = 0
  if accept was cancelled:
    resubmit multishot ACCEPT
```

## Periodic Stats Collection

```
multishot TIMEOUT (5 second interval, user_data = STATS_TIMER)

on STATS_TIMER CQE:
  snapshot ring metrics (PBUF_STATUS, in-flight count)
  update prometheus counters
  // no syscall needed for the timer itself
```

## Flags

| Flag | Effect |
|------|--------|
| `IORING_TIMEOUT_MULTISHOT` | Repeat until cancelled |
| `IORING_TIMEOUT_ABS` | Absolute time reference (combine with multishot for wall-clock alignment) |
| `IORING_TIMEOUT_BOOTTIME` | Use CLOCK_BOOTTIME (survives suspend) |
| `IORING_TIMEOUT_REALTIME` | Use CLOCK_REALTIME |
| `IORING_TIMEOUT_ETIME_SUCCESS` | Return 0 instead of -ETIME on expiry |

**`ETIME_SUCCESS` is essential for multishot.** Without it, every tick returns `-ETIME` which looks like an error. With it, CQE res is 0 on each tick.

## Registered Clock (6.13)

`IORING_REGISTER_CLOCK` sets a custom clock source for the ring. All timeouts use it. Combine with multishot for consistent timing across all timer SQEs without per-SQE clock flags.

## Precision

Multishot timeouts are **not** high-precision timers. They're subject to:
- Kernel timer resolution (typically 1ms with HZ=1000)
- Task work delivery latency (COOP_TASKRUN adds jitter)
- CQ consumption latency (your event loop speed)

For <1ms precision, use SQPOLL + IOPOLL. For wall-clock alignment, use absolute timeouts.

## vs Userspace Timers

| Approach | Syscalls | Precision | Integration |
|----------|----------|-----------|-------------|
| Multishot timeout | 0 after setup | ~1ms | Native CQ delivery |
| timerfd + POLL_ADD | 0 after setup | ~1ms | fd-based, needs poll |
| clock_gettime + sleep | 1 per tick | ~1ms | External to ring |
| SQPOLL + registered wait min_wait_usec | 0 | ~10μs | Implicit batching timer |

Multishot timeout is the right choice for application-level rate limiting and periodic tasks. For sub-millisecond timing, use min_wait_usec in registered wait instead.
