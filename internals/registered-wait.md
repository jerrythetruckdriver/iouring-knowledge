# Registered Wait Regions (6.13+)

`io_uring_reg_wait` — submit wait parameters through registered memory instead of syscall arguments.

## The Problem

Every `io_uring_enter()` with `IORING_ENTER_GETEVENTS` passes wait parameters through syscall args or a `struct io_uring_getevents_arg`. This means:

1. Kernel copies args from userspace each call
2. Can't batch wait configuration changes
3. Extra overhead on the hot path

## The Solution

Register a memory region containing wait parameters. The kernel reads directly from shared memory.

### Setup

```c
struct io_uring_reg_wait {
    struct __kernel_timespec ts;  // Timeout
    __u32 min_wait_usec;          // Minimum wait before returning
    __u32 flags;                  // Wait flags
    __u64 sigmask;                // Signal mask
    __u32 sigmask_sz;             // Signal mask size
    __u32 pad[3];
    __u64 pad2[2];
};

// Register the wait region
struct io_uring_reg_wait rw = { ... };
io_uring_register(ring_fd, IORING_REGISTER_CQWAIT_REG, &rw, 1);
```

### Usage

After registration, `io_uring_enter2()` with `IORING_ENTER_REG_WAIT` uses the registered region instead of reading args.

```c
// Instead of passing args each time:
io_uring_enter2(ring_fd, 0, 1,
    IORING_ENTER_GETEVENTS | IORING_ENTER_REG_WAIT,
    NULL, 0);  // Args come from registered region
```

## min_wait_usec

New with registered wait. Tells the kernel: "Don't return until at least this many microseconds have passed, even if a CQE is ready."

Use case: Batch completions. If your processing loop is cheap, waiting a few microseconds to collect more CQEs before returning to userspace reduces syscall frequency.

```c
rw.min_wait_usec = 50;  // Wait at least 50μs to batch CQEs
```

Trade-off: Latency vs. throughput. Higher values = more batching = fewer syscalls, but individual request latency increases.

## When to Use

- **High-frequency small I/O**: Reducing per-syscall overhead matters
- **Batch-oriented workloads**: min_wait_usec amortizes processing
- **SQPOLL alternative**: Get some batching benefits without dedicating a kernel thread

## When to Skip

- Low I/O frequency — overhead savings negligible
- Latency-sensitive paths — min_wait_usec adds delay
- Simple applications — added complexity not worth it
