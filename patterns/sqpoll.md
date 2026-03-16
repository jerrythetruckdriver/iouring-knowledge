# SQ Polling (SQPOLL)

Kernel thread polls the submission queue. You write SQEs, kernel picks them up. Zero syscalls for submission.

## Setup

```c
struct io_uring_params p = {
    .flags = IORING_SETUP_SQPOLL,
    .sq_thread_idle = 2000, // ms before thread sleeps
};

// Optional: pin to CPU
p.flags |= IORING_SETUP_SQ_AFF;
p.sq_thread_cpu = 3;
```

## The Wakeup Dance

SQ poll thread sleeps after `sq_thread_idle` ms of no submissions. Check the flag:

```c
io_uring_sqe_set_data(sqe, data);
io_uring_submit(ring); // liburing handles wakeup

// Manual:
if (*ring->sq.kflags & IORING_SQ_NEED_WAKEUP) {
    io_uring_enter(fd, 0, 0, IORING_ENTER_SQ_WAKEUP);
}
```

## When to Use SQPOLL

- High-frequency submission (>100K ops/sec)
- Latency-sensitive workloads where syscall overhead matters
- Storage workloads with `IOPOLL`

## When NOT to Use SQPOLL

- Low-frequency I/O (wastes CPU polling)
- Many rings (each gets a kernel thread)
- When `DEFER_TASKRUN` is sufficient

## SQPOLL + IOPOLL

The zero-syscall dream. SQ poll thread submits, I/O poll completes:

```c
params.flags = IORING_SETUP_SQPOLL | IORING_SETUP_IOPOLL;
```

Both submission and completion happen without syscalls. Pure shared-memory I/O. CPU cost is the polling threads.

## Cost

SQPOLL burns a full CPU core (or fraction thereof) on polling. `sq_thread_idle` controls the tradeoff — shorter idle means less latency spike on wakeup, longer idle saves CPU.

For most network workloads, `DEFER_TASKRUN` + `SINGLE_ISSUER` gives you 90% of the benefit at near-zero CPU cost.
