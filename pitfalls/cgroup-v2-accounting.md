# io_uring and cgroup v2 I/O Accounting

## How It Works

io_uring I/O is charged to the **cgroup of the task that created the ring**, not the task that submitted the SQE. This matters for:

- Container I/O limits (io.max, io.weight)
- I/O accounting (io.stat)
- Memory accounting (memory.current includes pinned pages)

## Block I/O Throttling

```
# Container cgroup v2 I/O limits
echo "259:0 rbps=10485760 wbps=10485760" > /sys/fs/cgroup/container/io.max
```

io_uring read/write operations respect these limits. The kernel associates the `bio` with the issuing task's blkcg when the request is processed.

### SQPOLL Caveat

With SQPOLL, the kernel thread submits I/O — but the blkcg association still tracks back to the ring creator. The `io_sq_thread` inherits the creating task's cgroup context.

### io-wq Workers

Same story. io-wq worker threads inherit the cgroup context of the task that created the ring. I/O from workers is correctly attributed.

## Memory Accounting

### Registered Buffers

Pages pinned via `IORING_REGISTER_BUFFERS` are accounted against:
- `RLIMIT_MEMLOCK` (per-user, pre-5.12 also per-ring)
- cgroup v2 `memory.current` (pinned pages count toward memory limit)

### Ring Memory

The SQ/CQ rings themselves are mmap'd memory. Accounted in cgroup memory.

### Provided Buffer Rings

Shared memory between kernel and userspace. Counted in memory accounting.

## I/O Statistics

```bash
cat /sys/fs/cgroup/container/io.stat
# 259:0 rbytes=1048576 wbytes=524288 rios=256 wios=128 dbytes=0 dios=0
```

io_uring operations appear in `rios`/`wios` counts. No special attribution — they look like normal block I/O.

## Gotchas

### 1. Ring Creation vs Submission

If process A creates a ring and passes the fd to process B (via SCM_RIGHTS), all I/O through that ring is charged to A's cgroup. This can confuse accounting in container orchestration.

### 2. NVMe Passthrough

`URING_CMD` to NVMe devices bypasses the block layer. Depending on the command, it may or may not be accounted by blk-cgroup. Standard read/write passthrough commands are accounted; admin commands may not be.

### 3. IOPOLL

With `IORING_SETUP_IOPOLL`, completions are polled by the submitting task. CPU time for polling is correctly attributed to the task's cgroup, but the busy-wait nature means `iowait` accounting differs from interrupt-driven I/O.

### 4. Memory Limits

If a container hits its memory limit, `IORING_REGISTER_BUFFERS` may fail with ENOMEM even if RLIMIT_MEMLOCK has room. Both limits must be satisfied.

## Kubernetes Implications

When running io_uring workloads in K8s pods:

```yaml
resources:
  limits:
    memory: "1Gi"   # Must account for registered buffers + ring memory
```

The memory limit must include:
- Ring memory (SQ + CQ + SQEs)
- Registered buffer pages (pinned, non-swappable)
- Provided buffer ring shared memory
- io-wq worker thread stacks

For a 256-entry ring with 64MB registered buffers:
- Ring: ~100KB
- Registered buffers: 64MB (pinned)
- Workers: ~24KB per thread
- Total memory overhead: ~64.2MB

## Best Practices

1. **Account for pinned memory** in container memory limits
2. **Use provided buffer rings** instead of registered buffers when possible (lower memory footprint per ring)
3. **One ring per container task** — avoid ring fd passing across cgroup boundaries
4. **Monitor io.stat** — io_uring I/O appears normally, no special tooling needed
5. **Test with io.max** — verify your io_uring workload respects container I/O throttling
