# Memory Pressure and io_uring

## TL;DR

io_uring pins memory. Under memory pressure, that pinned memory can't be reclaimed. Know what's pinned, size it conservatively, and handle ENOMEM gracefully.

## What Gets Pinned (Not Reclaimable)

| Resource | Memory per unit | Reclaimable? |
|----------|----------------|--------------|
| SQ ring + SQEs | 64B × entries (SQE) + overhead | No (mmap'd) |
| CQ ring + CQEs | 16B × entries (CQE) + overhead | No (mmap'd) |
| Registered buffers | Actual buffer pages (pinned) | No |
| Provided buffer rings | Ring header + buffer metadata | No (mmap'd) |
| Provided buffer data | User-allocated, not pinned | Yes (if not registered) |
| io_kiocb (per request) | ~280 bytes | Freed on completion |
| io-wq worker stacks | ~24KB per worker | Freed when worker exits |
| SQPOLL thread stack | ~24KB | Freed on ring teardown |

## ENOMEM Scenarios

### Ring Creation

```c
// io_uring_setup() can fail with ENOMEM when:
// - System is low on memory
// - RLIMIT_MEMLOCK exceeded (pre-5.12 kernels)
// - Requested ring size too large for available memory
int fd = io_uring_setup(entries, &params);
if (fd < 0 && errno == ENOMEM) {
    // Retry with smaller ring
    params.sq_entries = entries / 2;
    fd = io_uring_setup(params.sq_entries, &params);
}
```

### Buffer Registration

```c
// REGISTER_BUFFERS pins pages via get_user_pages_fast()
// Fails with ENOMEM when:
// - Pages can't be pinned (swapped out + no memory to bring back)
// - RLIMIT_MEMLOCK exceeded
// - Too many pinned pages system-wide
int ret = io_uring_register_buffers(ring, iovs, nr);
if (ret == -ENOMEM) {
    // Register fewer/smaller buffers
    // Or use provided buffer rings instead (not pinned)
}
```

### CQ Overflow Under Memory Pressure

When CQ is full and kernel needs to allocate overflow entries:

```c
// Kernel calls io_overflow_cqe() which heap-allocates
// Under memory pressure, this allocation can fail
// → Request is completed but CQE is LOST
// → Application sees IORING_SQ_CQ_OVERFLOW flag
// → After draining, overflow entries are freed
```

## OOM Killer Interaction

Pinned pages (registered buffers) count against the process's RSS but can't be reclaimed by the OOM killer. This means:

1. Process pins 8GB of registered buffers
2. System runs low on memory
3. OOM killer scores processes by RSS
4. Your io_uring process has high RSS (including pinned pages)
5. OOM killer targets your process — but killing it is the only way to free those pages

**Mitigation:** Set `oom_score_adj` appropriately for io_uring-heavy processes.

```bash
echo -500 > /proc/<pid>/oom_score_adj  # Less likely to be killed
```

## Low Memory Recovery Strategies

### 1. Size Rings Conservatively

```c
// Don't: allocate maximum ring size
io_uring_queue_init(32768, &ring, 0);  // 32K entries = ~2MB SQEs alone

// Do: start small, resize if needed (6.13+)
io_uring_queue_init(256, &ring, 0);
// Later, if load increases:
io_uring_register_resize_rings(&ring, 1024, 2048);
```

### 2. Prefer Provided Buffers Over Registered Buffers

Registered buffers pin physical pages. Provided buffer rings don't.

```c
// Registered buffers: pages pinned, can't be swapped
io_uring_register_buffers(ring, iovs, nr);  // RLIMIT_MEMLOCK applies

// Provided buffer ring: metadata pinned, buffer data is normal heap
io_uring_register_buf_ring(ring, &reg, 0);
// Buffer data at ring->bufs[i].addr is normal malloc'd memory
// Kernel only pins the ring header page
```

### 3. Monitor VmLck

```bash
grep VmLck /proc/<pid>/status
# VmLck:    524288 kB  ← this is your pinned memory

# Alert if approaching RLIMIT_MEMLOCK
ulimit -l  # current limit in KB
```

### 4. Handle EAGAIN on Submission

Under memory pressure, even SQE submission can fail:

```c
int ret = io_uring_submit(ring);
if (ret == -EAGAIN) {
    // Kernel couldn't allocate internal request (io_kiocb)
    // Drain some completions first, then retry
    io_uring_cq_advance(ring, processed);
    ret = io_uring_submit(ring);
}
```

### 5. Graceful Degradation

```c
// Fallback path when io_uring can't allocate
if (io_uring_submit() == -ENOMEM || io_uring_submit() == -EAGAIN) {
    // Fall back to synchronous I/O for this batch
    for (int i = 0; i < batch_size; i++) {
        pread(fd, buf, len, offset);  // Slow but works
    }
}
```

## cgroup Memory Limits

In containerized environments:

- Ring memory counts against `memory.current`
- Registered buffer pinned pages count against `memory.current`
- cgroup OOM killer can kill io_uring processes
- `memory.max` should account for pinned buffers

```
# K8s pod memory limit should include io_uring overhead
resources:
  limits:
    memory: "4Gi"  # app memory + ring memory + registered buffers
```

## Anti-Patterns

| Don't | Why |
|-------|-----|
| Register GBs of buffers on startup | Pins memory immediately, OOM risk |
| Ignore ENOMEM from register_buffers | Silent degradation, mysterious failures |
| Use huge CQ rings "just in case" | Wastes pinned memory |
| Skip VmLck monitoring | Can't diagnose memory pressure |
| Assume RLIMIT_MEMLOCK is unlimited | Varies by distro, container runtime |
