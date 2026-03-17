# IORING_ENTER_NO_IOWAIT

*Kernel 6.14+*

## The Problem

When io_uring submits I/O that goes to the block layer and waits for completion, the waiting time is classified as "iowait" in CPU accounting. This inflates the iowait metric visible in `top`, `mpstat`, and monitoring systems.

For applications doing heavy async I/O through io_uring, this creates misleading metrics. The CPU isn't actually stalled — it's efficiently waiting for I/O completions. But monitoring tools report high iowait, triggering false alerts.

## The Flag

```c
#define IORING_ENTER_NO_IOWAIT  (1U << 7)
```

Pass this to `io_uring_enter()` and the kernel won't account wait time as iowait. The CPU time gets classified as idle instead.

## Feature Detection

```c
#define IORING_FEAT_NO_IOWAIT  (1U << 17)
```

Check `io_uring_params.features` after `io_uring_setup()`. If `IORING_FEAT_NO_IOWAIT` is set, the kernel supports it.

## When to Use

- **Production monitoring** where iowait metrics trigger autoscaling or alerts
- **Mixed workloads** where io_uring I/O inflates CPU metrics for co-located services
- **Benchmarking** where you want clean CPU utilization numbers

## When NOT to Use

- If your monitoring specifically tracks iowait to detect storage bottlenecks
- If you need iowait attribution for capacity planning

## Usage

```c
io_uring_enter(ring_fd, to_submit, min_complete,
               IORING_ENTER_GETEVENTS | IORING_ENTER_NO_IOWAIT,
               NULL, 0);
```

Simple flag. No side effects beyond accounting.
