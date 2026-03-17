# io_uring and Real-Time / Embedded Systems

PREEMPT_RT, deadline scheduling, and whether io_uring makes sense outside server workloads.

## PREEMPT_RT Compatibility

PREEMPT_RT (merged mainline in 6.12) converts most kernel spinlocks to RT-mutexes, making kernel preemption points predictable. io_uring's interaction:

### What Works
- Basic submission/completion path is PREEMPT_RT compatible
- `DEFER_TASKRUN` + `SINGLE_ISSUER`: task work runs in the submitter's context, avoiding unpredictable kernel thread wakeups
- `COOP_TASKRUN`: no IPI interrupts — essential for RT, since IPIs cause unbounded latency spikes
- Registered resources: no per-op allocation, no memory pressure surprises

### What Doesn't (or is Risky)
- **SQPOLL**: The kernel poll thread runs at default priority. On PREEMPT_RT, it competes with RT tasks. You'd need to manually set its scheduling policy via `/proc/<pid>/sched`
- **io-wq workers**: Kernel threads with unpredictable wake patterns. On RT, these can cause priority inversion if they hold resources needed by RT tasks
- **CQ overflow**: Overflow list processing happens under a lock. On RT, this becomes an RT-mutex — bounded but adds latency
- **Memory allocation**: Some paths allocate memory (io-wq thread spawn, overflow handling). On RT, memory allocation latency is unbounded without `GFP_ATOMIC`

### RT-Safe Configuration
```c
IORING_SETUP_COOP_TASKRUN |
IORING_SETUP_DEFER_TASKRUN |
IORING_SETUP_SINGLE_ISSUER |
IORING_SETUP_NO_SQARRAY |
IORING_SETUP_SUBMIT_ALL
```

No SQPOLL, no IOPOLL. Pre-register everything. Size CQ ring large enough to never overflow. Avoid operations that trigger io-wq offload (keep everything inline).

## Deadline Scheduling

Linux's `SCHED_DEADLINE` provides guaranteed CPU time within a period. io_uring's SQPOLL thread doesn't natively support deadline scheduling, but:

```bash
# After identifying the SQPOLL thread PID
chrt -d --sched-runtime 1000000 --sched-deadline 5000000 --sched-period 5000000 $SQPOLL_PID
```

This gives the SQPOLL thread 1ms of guaranteed CPU every 5ms. Enough for submission polling without starving other tasks.

**Problem**: SQPOLL thread PID isn't directly exposed. You find it by scanning `/proc` for `iou-sqp-*` threads.

## Embedded Linux Use Cases

### Storage I/O on Resource-Constrained Systems

Embedded systems with eMMC/NVMe and single-core or dual-core CPUs. io_uring helps by:
- Batching I/O reduces syscall overhead (matters more on slower cores)
- `NO_SQARRAY` + `SQ_REWIND` reduces ring memory footprint
- Small ring sizes (16-32 entries) are fine for embedded workloads

### Sensor Data Collection

Multiple sensor fds (SPI, I2C character devices, network sockets) multiplexed with io_uring:

```c
// One ring handles all sensors
// Multishot read on each sensor fd
for (sensor : sensors) {
    sqe = io_uring_get_sqe(ring);
    io_uring_prep_read_multishot(sqe, sensor->fd, ...);
}
```

Fewer threads than epoll+thread-pool. Lower memory footprint. Better cache behavior on small L1/L2.

### Robot Control Loops

Fixed-rate control loops (1KHz-10KHz) need deterministic I/O:

```c
// Linked timeout ensures bounded latency
sqe = prep_read(sensor_fd, ...);
sqe->flags |= IOSQE_IO_LINK;

sqe = prep_link_timeout(1_ms);  // abort if sensor doesn't respond in 1ms

io_uring_submit(ring);
```

If the read doesn't complete within the link timeout, it gets cancelled. Bounded worst-case latency.

## What's Missing for Embedded/RT

1. **Priority-aware submission**: No way to mark SQEs as high/low priority. All requests are equal.
2. **Deadline-aware worker pool**: io-wq doesn't respect `SCHED_DEADLINE` or `SCHED_FIFO` natively.
3. **Memory pre-allocation**: Some code paths still do runtime allocation. A "no-alloc" mode would help RT.
4. **Guaranteed inline completion**: No way to force an operation to complete inline (IOSQE_ASYNC forces io-wq; the opposite doesn't exist).

## Should You Use io_uring on Embedded?

**Yes, if**:
- You have Linux ≥ 5.19 (for COOP_TASKRUN)
- Your workload has multiple concurrent I/O operations
- You need lower syscall overhead on slow CPUs
- You can pre-register all resources and avoid io-wq paths

**No, if**:
- Single-threaded, single-fd workload (just use blocking I/O)
- Kernel is too old (< 5.1)
- Hard real-time guarantees needed at sub-millisecond level (use RTOS)
- Your embedded Linux kernel has io_uring disabled (`kernel.io_uring_disabled=2`)

## vs Other Approaches on Embedded

| Approach | Overhead | Determinism | Complexity |
|----------|----------|-------------|------------|
| Blocking I/O + threads | High (context switches) | Poor | Low |
| epoll + non-blocking | Medium (2 syscalls/event) | OK | Medium |
| io_uring (careful config) | Low (batched syscalls) | Good | Higher |
| Direct register access | Zero | Best | Highest |

io_uring sits in the sweet spot between "easy but slow" and "fast but hardware-specific." Good for embedded Linux systems that aren't doing bare-metal register-banging.
