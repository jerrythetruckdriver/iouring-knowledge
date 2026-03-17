# io_uring for Latency-Sensitive Workloads

Games, audio, trading systems — anything where p99 latency matters more than throughput.

## The Latency Problem

Traditional async I/O sources of jitter:
1. **Syscall overhead** — context switch to kernel and back (~1-5µs)
2. **Interrupt coalescing** — hardware batches interrupts, adds latency
3. **Scheduler preemption** — your I/O completion handler gets descheduled
4. **Memory allocation** — per-I/O alloc in the hot path

io_uring addresses all four.

## Low-Latency Configuration

```c
struct io_uring_params p = {
    .flags = IORING_SETUP_SQPOLL        // (1) eliminate submission syscalls
           | IORING_SETUP_IOPOLL        // (2) poll for completions, no interrupts
           | IORING_SETUP_SINGLE_ISSUER // reduce atomic contention
           | IORING_SETUP_COOP_TASKRUN  // (3) avoid IPI reschedules
           | IORING_SETUP_NO_SQARRAY,   // flat submission, less indirection
    .sq_thread_idle = 1000,             // 1ms before SQPOLL sleeps
};
```

### (1) SQPOLL: Zero Submission Syscalls

Kernel thread polls the SQ. You write SQEs to shared memory, kernel picks them up. No `io_uring_enter()` needed for submission.

**Cost:** Dedicates a CPU core. Worth it only if you're submitting continuously.

### (2) IOPOLL: Polling Completions

No interrupt handler. Kernel actively polls the device for completions. Eliminates interrupt latency (~10-50µs savings on NVMe).

**HYBRID_IOPOLL (6.13):** Sleeps briefly, then polls. Better power efficiency, slightly higher latency than strict IOPOLL.

### (3) COOP_TASKRUN: No IPI

Normally, when an I/O completes on another CPU, it sends an Inter-Processor Interrupt to wake your task. COOP_TASKRUN defers this — work runs when your task naturally transitions to kernel.

**DEFER_TASKRUN (6.1):** Even more aggressive — defers ALL task work to `io_uring_enter()`. Maximum control over when completions are processed.

## Registered Resources

Pre-register everything to eliminate per-I/O setup:

```c
// Register buffers — avoid per-I/O DMA mapping
io_uring_register(fd, IORING_REGISTER_BUFFERS, iovs, nr);

// Register files — avoid per-I/O fdget/fdput
io_uring_register(fd, IORING_REGISTER_FILES, fds, nr);

// Register clock — consistent timing source
struct io_uring_clock_register clk = { .clockid = CLOCK_MONOTONIC_RAW };
io_uring_register(fd, IORING_REGISTER_CLOCK, &clk, 1);

// Register wait args — zero-alloc completion wait
// (via IORING_REGISTER_MEM_REGION)
```

## Registered Wait (6.13)

The fastest completion wait path:

```c
struct io_uring_reg_wait w = {
    .ts = { .tv_sec = 0, .tv_nsec = 100000 }, // 100µs timeout
    .min_wait_usec = 10,  // batch completions for 10µs
    .flags = IORING_REG_WAIT_TS,
};
// Pre-registered in memory region, indexed by slot
io_uring_enter(fd, 0, 1, IORING_ENTER_GETEVENTS | IORING_ENTER_EXT_ARG_REG, slot);
```

`min_wait_usec` is the key: wait up to N microseconds for more completions before returning. Trades a tiny latency increase for fewer wakeups.

## Audio / Real-Time Patterns

For audio processing (~5ms budget per buffer):

1. Pre-allocate all buffers, register them
2. Use linked read → process → write chains
3. SQPOLL if buffer period < 10ms
4. NO_IOWAIT (6.14) to avoid polluting iowait stats
5. Pin SQPOLL thread to dedicated core with `IORING_SETUP_SQ_AFF`

## What io_uring Can't Fix

- **Kernel scheduling:** Still subject to CFS/EEVDF scheduling. Use SCHED_FIFO/SCHED_DEADLINE for the I/O thread.
- **Page faults:** If your buffers get swapped out, you lose. Use `mlock()`.
- **NUMA effects:** Ring and buffers should be on the same NUMA node as the I/O thread.
- **Not RTOS:** Linux is not a real-time OS. Even with PREEMPT_RT + io_uring, worst-case latency is unbounded (just very unlikely to be bad).

## vs DPDK for Trading Systems

DPDK gives lower median latency (kernel bypass). io_uring gives:
- Standard socket API compatibility
- No dedicated NIC / driver requirements  
- Works with any block device, not just networking
- Composable with other I/O (disk + network in one ring)

For sub-microsecond networking: DPDK. For mixed I/O with low latency: io_uring.
