# Cross-Ring Buffer Sharing

## The Problem

Thread-per-core architectures use one ring per thread. Registered buffers are per-ring. Without sharing, each ring pins its own copy of the buffer pool — memory scales linearly with thread count.

## IORING_REGISTER_CLONE_BUFFERS (6.13)

Clone a registered buffer table from one ring to another:

```c
struct io_uring_clone_buffers args = {
    .src_fd  = source_ring_fd,
    .flags   = 0,
    .src_off = 0,
    .dst_off = 0,
    .nr      = 0,  // 0 = clone all
};
io_uring_register(dst_ring_fd, IORING_REGISTER_CLONE_BUFFERS, &args, 0);
```

After cloning, both rings reference the same physical pages. The buffers are pinned once, charged once to RLIMIT_MEMLOCK.

## Flags

```c
// Source ring's buffers are already registered (use registered fd)
IORING_REGISTER_SRC_REGISTERED  (1 << 0)

// Replace destination's existing buffer table
IORING_REGISTER_DST_REPLACE     (1 << 1)
```

## Partial Clone

Clone a subset of buffers:

```c
struct io_uring_clone_buffers args = {
    .src_fd  = source_ring_fd,
    .flags   = 0,
    .src_off = 0,    // Start at source index 0
    .dst_off = 0,    // Place at dest index 0
    .nr      = 64,   // Clone first 64 buffers only
};
```

## Patterns

### Thread-Per-Core Setup

```
Main thread:
  1. Create ring_0
  2. Register buffers on ring_0 (large pool, pinned once)
  
Worker threads 1..N:
  1. Create ring_N
  2. Clone buffers from ring_0
  
Result: N+1 rings share one buffer pool
```

Memory savings: N threads × buffer_pool_size → 1 × buffer_pool_size. For 16 threads with 256MB buffer pool, that's 4GB → 256MB.

### Hot-Swap With Clone

Replace buffers at runtime without stopping I/O:

```
1. Register new buffers on a temporary ring
2. Drain in-flight ops on target ring
3. Clone from temp ring with IORING_REGISTER_DST_REPLACE
4. Close temp ring
```

### Asymmetric Buffers

Clone different subsets to different rings:

```
ring_0: owns all buffers (indexes 0-1023)
ring_1: clone indexes 0-255 (network I/O)
ring_2: clone indexes 256-511 (disk I/O)
ring_3: clone indexes 512-1023 (scratch space)
```

## Provided Buffer Rings (Different)

Clone buffers are for *registered* buffers (IORING_OP_READ_FIXED, IORING_OP_WRITE_FIXED). Provided buffer rings (REGISTER_PBUF_RING) are separate — they're per-ring shared memory and can't be cloned.

For provided buffers across rings, the application shares the memory region manually:

```c
// Both rings register pbuf rings pointing to the same mmap'd region
// Application manages concurrency (each ring gets its own bgid range)
```

This is safe because provided buffer rings are SPSC — one producer (application), one consumer (kernel per ring). Give each ring its own buffer group ID, even if the underlying memory overlaps.

## MSG_RING + Buffers

Pass buffer *references* between rings using MSG_RING:

```c
// Ring A completed a recv into buffer group, got buf_id=42
// Forward to Ring B:
io_uring_prep_msg_ring(sqe, ring_b_fd, result_len, encode(buf_id, 42));
```

Ring B decodes the user_data, accesses the shared buffer memory, processes it, then returns the buffer to the pool. Application-level buffer lifecycle management.

## Constraints

1. **Clone copies the table, not a reference.** After clone, registering new buffers on the source doesn't affect the clone. Re-clone if the source table changes.
2. **Can't clone to a ring that has in-flight ops using registered buffers.** Drain first.
3. **Clone requires both rings in the same process.** For cross-process, use shared memory + independent registration.
4. **RLIMIT_MEMLOCK charged once.** The physical pages are pinned once regardless of how many rings clone them.

## When to Use What

| Scenario | Mechanism |
|----------|-----------|
| Shared read-only buffers across threads | Clone buffers |
| Per-connection recv buffers | Provided buffer rings (per-ring) |
| Forwarding data between rings | MSG_RING + shared memory |
| Hot-swap buffer pool | Clone with DST_REPLACE |
| Cross-process buffer sharing | Shared memory + independent registration |
