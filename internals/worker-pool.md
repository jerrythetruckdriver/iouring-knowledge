# io_uring Worker Pool (io-wq)

io_uring isn't just a submission queue. It's a runtime with its own thread pool: `io-wq`.

Source: Cloudflare's investigation + kernel source (`fs/io-wq.c`).

## Two Types of Workers

| Type | Handles | Default Limit |
|------|---------|---------------|
| **Bounded** | Regular files (`S_IFREG`), block devices (`S_ISBLK`) | `f(SQ_size, nr_cpus)` |
| **Unbounded** | Everything else — sockets, char devices, pipes | `RLIMIT_NPROC` |

Workers are spawned via `create_io_thread()` → `create_io_worker()` in kernel space.

## When Workers Spawn

io_uring is smart about it:

1. **Non-blocking first**: For socket ops, tries non-blocking I/O. If `-EAGAIN`, registers a poll wakeup instead of spawning a thread.
2. **IOSQE_ASYNC**: Forces async dispatch — skips the non-blocking attempt, goes straight to worker pool. This is how you end up with thousands of threads.
3. **Blocking required**: If the operation can't do non-blocking (e.g., `open()` on regular file), dispatched to bounded worker.

Tracepoint flow:
```
submit_sqe ──▶ try non-blocking ──▶ -EAGAIN ──▶ poll_arm (no worker)
                                                    │
submit_sqe ──▶ IOSQE_ASYNC ──▶ queue_async_work ──▶ worker thread
```

## Configuring Worker Limits

### IORING_REGISTER_IOWQ_MAX_WORKERS (5.15+)

Best method. Per-ring, per-NUMA-node.

```c
unsigned int workers[2] = {
    0,   // [0] = bounded workers (0 = don't change)
    32   // [1] = unbounded workers
};
io_uring_register(ring_fd, IORING_REGISTER_IOWQ_MAX_WORKERS, workers, 2);
// On return, workers[] contains previous values
```

### RLIMIT_NPROC

Overrides IOWQ_MAX_WORKERS. **Dangerous** if shared UID:

- io_uring retries `create_io_thread()` when work queues, gets `-EAGAIN`
- Retry loop burns 100% CPU on one core
- ~300k+ failed spawn attempts per second observed

Fix: use a dedicated UID or user namespace.

### cgroup pids.max

Same problem as RLIMIT_NPROC — io_uring doesn't back off gracefully when `create_io_thread()` fails due to cgroup limits.

## Worker Pool Scope

**Per-thread, not per-ring.**

```
Thread A → ring1, ring2 → shared worker pool (max_workers limit applies once)
Thread B → ring3        → separate worker pool (own max_workers)
```

Single-threaded process with 2 rings, workers=2: **2 total workers**.
Multi-threaded process with 2 threads × 1 ring, workers=2: **4 total workers** (2 per thread).

Workers are named `iou-wrk-<owning_tid>`.

## NUMA Awareness

Worker pools are per-NUMA-node. `IORING_REGISTER_IOWQ_MAX_WORKERS` sets the limit per node.

Internal structure:
```
task_struct
  └─ io_uring_task
       └─ io_wq
            └─ io_wqe[nr_numa_nodes]
                 └─ io_wqe_acct[2]  // bounded, unbounded
                      ├─ max_workers
                      └─ nr_workers
```

On a 2-node NUMA system with unbounded workers=8: up to 16 total workers (8 per node).

## Worker Lifecycle

1. **Spawn**: `create_io_thread()` when work queued and `nr_workers < max_workers`
2. **Run**: Worker picks items from work queue, executes blocking I/O
3. **Idle**: After completing work, waits briefly for more
4. **Retire**: No pending work after grace period → thread exits

## Pitfalls

1. **Don't rely on RLIMIT_NPROC alone** — CPU burn on spawn failure
2. **Always set IORING_REGISTER_IOWQ_MAX_WORKERS** — explicit is better than default
3. **Multi-threaded apps multiply workers** — each thread gets its own pool
4. **IOSQE_ASYNC on sockets** = one worker per request. Usually wrong — let io_uring use poll.
5. **Container deployments**: If cgroup pids.max is low, io_uring will spin. Set IOWQ_MAX_WORKERS below the cgroup limit.

## Monitoring

```bash
# Worker count
pstree -pt <pid> | grep -c iou-wrk

# Failed spawns (CPU burn indicator)
sudo bpftrace --btf -e 'kr:create_io_thread { @[retval] = count(); } i:s:1 { print(@); clear(@); }'

# Async dispatch rate
sudo perf stat -e io_uring:io_uring_queue_async_work -- sleep 5
```
