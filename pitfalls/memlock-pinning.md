# Memory Pinning and RLIMIT_MEMLOCK

## The Problem

io_uring pins memory. Pinned memory can't be swapped, which means it counts against resource limits and consumes physical RAM. Get this wrong and you hit `ENOMEM` or `EPERM` at the worst possible time.

## What Gets Pinned

### Ring Memory
- **SQ ring**: `sq_entries * sizeof(u32)` (SQ array) + overhead ≈ few KB
- **CQ ring**: `cq_entries * sizeof(struct io_uring_cqe)` = entries × 16 bytes (or 32 with CQE32)
- **SQEs**: `sq_entries * sizeof(struct io_uring_sqe)` = entries × 64 bytes (or 128 with SQE128)
- Default 4096-entry ring: ~420KB total

### Registered Buffers
Each `iov_base` region gets its pages pinned via `pin_user_pages_fast()`.

```
Cost per registered buffer = ALIGN(len, PAGE_SIZE) / PAGE_SIZE * sizeof(struct page *)
                           + actual pinned pages (physical memory)
```

10 buffers × 64KB each = 640KB of pinned physical memory + ~160 page struct pointers.

### Registered Files
Minimal overhead: ~8 bytes per slot in the fixed file table. Not memory-pinned, just kernel-side fd references.

### Provided Buffer Rings
The buffer ring itself is shared memory (mmap'd or user-provided). Each buffer's backing memory is **not** pinned until actually used in an I/O operation. Light on memlock.

## RLIMIT_MEMLOCK Accounting

### Pre-5.12: Strict Accounting
All io_uring memory counted against `RLIMIT_MEMLOCK`. This meant:
- Default 64KB limit on most distros → couldn't even create a ring
- Had to bump `ulimit -l` or run as root

### 5.12+: IORING_SETUP_R_DISABLED + Relaxed Accounting
Ring memory itself no longer counted against RLIMIT_MEMLOCK in most configurations. The kernel uses `__GFP_ACCOUNT` for cgroup memory accounting instead.

### Current Behavior (6.x)
- **Ring memory**: Uses kernel memory accounting (memcg), not RLIMIT_MEMLOCK
- **Registered buffers**: Still pin pages, counted against the task's locked memory. `RLIMIT_MEMLOCK` applies.
- **Registered files**: No RLIMIT_MEMLOCK impact
- **Provided buffer rings**: Shared mapping, RLIMIT_MEMLOCK applies to the mmap

## Common Failures

### ENOMEM on REGISTER_BUFFERS
```
io_uring_register(fd, IORING_REGISTER_BUFFERS, iovs, nr) = -ENOMEM
```

Causes:
1. `RLIMIT_MEMLOCK` too low for total pinned size
2. System running low on physical memory (can't pin what doesn't exist)
3. Too many pages to pin (kernel has internal limits)

Fix:
```bash
# Check current limit
ulimit -l

# Increase (per-session)
ulimit -l unlimited

# Persistent (/etc/security/limits.conf)
myuser  soft  memlock  unlimited
myuser  hard  memlock  unlimited

# Or systemd service
[Service]
LimitMEMLOCK=infinity
```

### EPERM on Ring Creation
Pre-5.12 kernels + unprivileged user + default memlock limit. Upgrade kernel or raise limit.

### Silent OOM
Pinning 100 × 1MB buffers = 100MB of non-swappable physical RAM. On a 4GB system with other processes, this can trigger OOM killer — not on your process, but on someone else's. No warning.

## Production Sizing

### Conservative Formula
```
memlock_needed = ring_memory
               + sum(registered_buffer_sizes)
               + provided_buffer_ring_size
               + safety_margin(20%)
```

### Examples

**Web server (1000 connections, 4KB buffers)**:
- Ring: 4096 entries → ~420KB
- Registered buffers: none (use provided buffers)
- Provided buffer ring: 4096 × 4KB = 16MB shared mapping
- RLIMIT_MEMLOCK needed: ~20MB

**Database (O_DIRECT, 256KB aligned buffers)**:
- Ring: 1024 entries → ~100KB
- Registered buffers: 64 × 256KB = 16MB pinned
- RLIMIT_MEMLOCK needed: ~20MB

**High-throughput proxy (many rings, thread-per-core)**:
- 16 cores × 1 ring each = 16 × 420KB = ~7MB
- Clone buffers between rings (shared pinned pages)
- RLIMIT_MEMLOCK needed: ~50MB + shared buffer pool

## Best Practices

1. **Use provided buffer rings** over registered buffers when possible. Provided buffers are shared memory, not pinned per-buffer. More memory-efficient for many small buffers.

2. **Clone buffers** between rings (`IORING_REGISTER_CLONE_BUFFERS`) instead of registering the same buffers on multiple rings. Shares the pinned pages.

3. **Register only hot buffers**. Don't register every buffer — register the ones in your fast path. Unregistered buffers still work, just without the kernel shortcut.

4. **Monitor with `/proc/<pid>/status`**:
   ```
   VmLck:     16384 kB    # Currently locked memory
   ```

5. **Set memlock limits explicitly** in systemd units or deployment configs. Don't rely on system defaults.

6. **Use `IORING_REGISTER_BUFFERS2`** (tagged registration) for dynamic updates without re-registering everything.

## cgroup v2 Interaction

With cgroup v2, memory accounting tracks kernel memory allocations including io_uring ring memory. A container with a 512MB memory limit will count ring memory against that limit, even if RLIMIT_MEMLOCK is unlimited.

Set `memory.max` appropriately in cgroup configs when using io_uring in containers.

## Platform Notes

- **Docker**: Default seccomp blocks io_uring entirely (separate issue from memlock)
- **Kubernetes**: Resource limits don't translate to RLIMIT_MEMLOCK — use security contexts
- **RHEL 9**: io_uring disabled by default via sysctl, memlock is moot
- **Systemd**: `LimitMEMLOCK=infinity` in service file, or use `MemoryLock=` in slice
