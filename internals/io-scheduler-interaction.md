# I/O Scheduler Interaction

## How io_uring Meets the Block Layer

io_uring submits I/O requests through the standard block layer. The I/O scheduler (mq-deadline, BFQ, kyber, none) sits between the block layer and the device driver. io_uring doesn't bypass it — unless you use IOPOLL or NVMe passthrough.

## Request Path

```
io_uring SQE → VFS → filesystem → block layer → I/O scheduler → device driver
```

With O_DIRECT, skip VFS page cache:
```
io_uring SQE → block layer → I/O scheduler → device driver
```

With IOPOLL:
```
io_uring SQE → block layer → I/O scheduler (none) → NVMe driver → polled completion
```

With NVMe passthrough (URING_CMD):
```
io_uring SQE → NVMe driver directly (no block layer, no scheduler)
```

## Scheduler Behavior

### none (noop)

FIFO passthrough. Best for NVMe devices where the firmware handles scheduling. Works with IOPOLL. Lowest overhead.

### mq-deadline

Merges and reorders requests. Adds latency but improves throughput for rotational devices. io_uring requests enter like any other block request — no special treatment. Deadline guarantees prevent starvation.

### BFQ (Budget Fair Queueing)

Per-process I/O scheduling with latency guarantees. io_uring requests are attributed to the submitting task (or the SQPOLL thread if used — this can distort BFQ's per-process fairness).

**SQPOLL + BFQ gotcha:** All I/O from the SQPOLL thread looks like one process to BFQ. If multiple applications share an SQPOLL ring via `IORING_SETUP_ATTACH_WQ`, BFQ can't distinguish them. Use separate rings or avoid BFQ with SQPOLL.

### kyber

Token-based scheduler targeting latency percentiles. Works fine with io_uring. No special interaction.

## IOPOLL Requirements

`IORING_SETUP_IOPOLL` requires:
1. O_DIRECT files (no buffered I/O)
2. Scheduler set to `none` (polled completions don't work with schedulers that reorder requests)
3. Device driver supports polling (NVMe, some virtio-blk)

```bash
echo none > /sys/block/nvme0n1/queue/scheduler
```

Hybrid IOPOLL (`IORING_SETUP_HYBRID_IOPOLL`, 6.13) relaxes this — it sleeps briefly before polling, works with `none` scheduler only.

## Request Merging

The block layer merges adjacent requests before they reach the device. io_uring benefits from this automatically:

- Linked sequential reads/writes get merged into larger I/O
- Batch submission increases merging opportunities
- The scheduler sees io_uring requests no differently than sync I/O

## Priority Support

io_uring supports I/O priority via `sqe->ioprio`:

```c
sqe->ioprio = IOPRIO_PRIO_VALUE(IOPRIO_CLASS_RT, 0);  // realtime
sqe->ioprio = IOPRIO_PRIO_VALUE(IOPRIO_CLASS_BE, 4);  // best-effort, priority 4
sqe->ioprio = IOPRIO_PRIO_VALUE(IOPRIO_CLASS_IDLE, 0); // idle
```

mq-deadline and BFQ honor these. `none` ignores them. The priority is per-SQE, so you can mix priorities in a single submission batch.

## cgroup I/O Accounting

io_uring I/O is charged to the submitting task's cgroup (blkcg). With SQPOLL, it's charged to the cgroup of the ring's creating task (not the SQPOLL thread). This matters for container I/O limits.

## Recommendations

| Workload | Device | Scheduler | io_uring Config |
|----------|--------|-----------|-----------------|
| Database (random) | NVMe | none | IOPOLL + O_DIRECT |
| Web server (mixed) | NVMe | none | Default or SQPOLL |
| File server (sequential) | HDD | mq-deadline | Default |
| Multi-tenant | NVMe | BFQ | Separate rings per tenant |
| Latency-critical | NVMe | none | IOPOLL + SQPOLL + registered everything |
| NVMe passthrough | NVMe | N/A | URING_CMD (bypasses scheduler) |

The scheduler choice matters more than the io_uring configuration. For NVMe, `none` is almost always correct. io_uring just submits requests — the scheduler decides what order they hit the device.
