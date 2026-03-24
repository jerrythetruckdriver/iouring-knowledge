# CPU Affinity: SQPOLL, Worker Threads, and NUMA

## SQPOLL Thread Pinning

When using `IORING_SETUP_SQPOLL`, the kernel creates a dedicated thread that polls the SQ for new entries.

### Pinning to a CPU

```c
struct io_uring_params p = {
    .flags = IORING_SETUP_SQPOLL | IORING_SETUP_SQ_AFF,
    .sq_thread_cpu = 3,    // Pin SQPOLL thread to CPU 3
    .sq_thread_idle = 2000, // Idle timeout in ms
};
```

**Why pin**: Without `IORING_SETUP_SQ_AFF`, the SQPOLL thread runs wherever the scheduler puts it. On NUMA systems, this can mean cross-node memory access to the SQ ring, destroying the low-latency benefit.

**Best practice**: Pin SQPOLL thread to a CPU on the same NUMA node as the ring memory and registered buffers.

### SQPOLL + Hyper-Threading

Avoid sharing a physical core between the SQPOLL thread and your application thread. If your app thread is on CPU 3 (HT sibling of CPU 7), pin SQPOLL to a different physical core — not CPU 7.

SQPOLL busy-polls, consuming 100% of its logical core. Sharing a physical core with your app thread means both compete for execution resources.

## io-wq Worker Affinity

io-wq workers handle operations that can't complete inline (blocking file I/O on non-O_DIRECT files, operations forced async with IOSQE_ASYNC).

### Default Behavior

Workers inherit the cpumask of the task that created the ring. If your main thread has affinity to CPUs 0-3, io-wq workers will be scheduled on CPUs 0-3.

### Explicit Control

```c
// Set io-wq worker affinity
cpu_set_t mask;
CPU_ZERO(&mask);
CPU_SET(4, &mask);
CPU_SET(5, &mask);
io_uring_register(ring_fd, IORING_REGISTER_IOWQ_AFF, &mask, sizeof(mask));
```

```c
// Reset to default
io_uring_register(ring_fd, IORING_UNREGISTER_IOWQ_AFF, NULL, 0);
```

**Use case**: Isolate io-wq workers from your hot-path CPUs. Put them on a separate NUMA node or dedicated cores.

### Worker Count Limits

```c
unsigned int workers[2] = { 4, 2 }; // bounded=4, unbounded=2
io_uring_register(ring_fd, IORING_REGISTER_IOWQ_MAX_WORKERS, workers, 2);
```

- **Bounded**: For I/O that should complete quickly (file I/O). Defaults to 4 × num_cpus.
- **Unbounded**: For potentially long operations. Defaults to RLIMIT_NPROC.

## NUMA Considerations

### Ring Memory Placement

By default, `io_uring_setup()` allocates ring memory on the node where the calling thread runs. With `IORING_SETUP_NO_MMAP`, you provide the memory — use `mbind()` or `mmap()` with NUMA policy:

```c
void *ring_mem = mmap(NULL, ring_size, PROT_READ | PROT_WRITE,
                      MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
mbind(ring_mem, ring_size, MPOL_BIND, &nodemask, maxnode, 0);
```

### Registered Buffer Placement

Registered buffers are pinned via `get_user_pages()`. The kernel pins whatever physical pages back the virtual addresses. For NUMA-local I/O:

```c
// Allocate on specific NUMA node before registering
void *buf = numa_alloc_onnode(buf_size, target_node);
struct iovec iov = { .iov_base = buf, .iov_len = buf_size };
io_uring_register_buffers(ring, &iov, 1);
```

### Thread-Per-Core NUMA Pattern

```
Node 0 (CPUs 0-7):
  Thread 0 → Ring 0 (CPU 0, SQPOLL on CPU 1)
    - Registered buffers on Node 0
    - io-wq workers on CPUs 2-3
  Thread 1 → Ring 1 (CPU 4, SQPOLL on CPU 5)
    - Registered buffers on Node 0
    - io-wq workers on CPUs 6-7

Node 1 (CPUs 8-15):
  Thread 2 → Ring 2 (CPU 8, SQPOLL on CPU 9)
    - Registered buffers on Node 1
    ...
```

### Cross-Node Penalties

| Access Pattern | Penalty |
|---------------|---------|
| Local DRAM | Baseline |
| Remote DRAM (1 hop) | ~50-100ns extra |
| Remote DRAM (2 hops) | ~100-200ns extra |

For io_uring specifically:
- **SQ/CQ access**: Application writes SQEs, kernel reads them. If on different nodes: penalty on every submission.
- **Registered buffers**: DMA targets. NIC/NVMe DMA to remote node memory adds latency.
- **Provided buffer rings**: Shared between app and kernel. Same-node placement critical.

## Monitoring

### Check SQPOLL Thread Placement

```bash
# Find SQPOLL thread
ps -eLo pid,tid,comm,psr | grep iou-sqp
# psr = processor (CPU) currently running on

# Check affinity
taskset -p <tid>
```

### Check io-wq Workers

```bash
ps -eLo pid,tid,comm,psr | grep iou-wrk
```

### NUMA Stats

```bash
numastat -p <pid>
# Look for numa_miss and numa_foreign — high values = cross-node access
```

## Common Mistakes

1. **SQPOLL without SQ_AFF**: Thread migrates between CPUs, cache-cold on every migration.
2. **Registered buffers on wrong node**: DMA goes cross-NUMA. Use `numactl --membind` or `mbind()`.
3. **io-wq workers competing with app threads**: Set explicit affinity to isolate them.
4. **Too many SQPOLL threads**: Each consumes a full CPU. One per ring, shared rings where possible.
5. **Ignoring HT siblings**: SQPOLL + app on same physical core = contention.
