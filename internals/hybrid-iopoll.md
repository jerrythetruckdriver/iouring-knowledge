# Hybrid IO Polling (IORING_SETUP_HYBRID_IOPOLL)

*Kernel 6.13+*

## The Problem with Strict IOPOLL

`IORING_SETUP_IOPOLL` makes the kernel busy-poll for completions instead of using interrupts. Great for ultra-low latency on NVMe. Terrible for power consumption and CPU utilization when the device isn't saturated.

Strict IOPOLL spins immediately after submission. If the device takes 10µs to complete and you start spinning at t=0, you burn 10µs of CPU doing nothing.

## Hybrid IOPOLL

`IORING_SETUP_HYBRID_IOPOLL` (bit 17) introduces an initial sleep delay before spinning.

```c
#define IORING_SETUP_HYBRID_IOPOLL  (1U << 17)
```

The kernel sleeps briefly after submission, then switches to polling. This captures most of IOPOLL's latency benefit while dramatically reducing CPU waste on devices that aren't sub-microsecond.

### How It Works

1. SQE submitted with IOPOLL ring
2. Instead of immediately spinning, kernel sleeps for a calibrated delay
3. After the sleep, switches to busy-poll mode
4. Polls until completion arrives

The sleep duration is adaptive — the kernel tracks device completion latency and adjusts.

### When to Use

| Scenario | Recommendation |
|----------|---------------|
| Ultra-fast NVMe (Optane, CXL) | Strict IOPOLL — device is fast enough that sleep adds latency |
| Standard NVMe SSD | **Hybrid IOPOLL** — best power/latency tradeoff |
| SATA/HDD | Neither — use interrupt-driven I/O |
| Mixed fast/slow devices | Hybrid IOPOLL — adapts to actual latency |

### Setup

```c
struct io_uring_params p = {
    .flags = IORING_SETUP_HYBRID_IOPOLL,
};
int fd = io_uring_setup(entries, &p);
```

Note: `HYBRID_IOPOLL` implies IOPOLL behavior. The ring still requires direct/O_DIRECT I/O and does not support interrupt-driven completion.

### vs. SQPOLL + IOPOLL

Both reduce syscall overhead for polled I/O. The difference:

- **SQPOLL + IOPOLL**: Kernel thread handles both submission and polling. Always spinning.
- **Hybrid IOPOLL**: App still calls `io_uring_enter()`, but the poll phase is smarter about when to spin.

You can combine them (`SQPOLL | HYBRID_IOPOLL`) for kernel-thread submission with adaptive polling.

## Feature Detection

Check `IORING_SETUP_HYBRID_IOPOLL` support via probe or trial setup. Kernels < 6.13 will return `-EINVAL`.
