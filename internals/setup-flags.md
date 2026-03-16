# Setup Flags

Flags passed to `io_uring_setup()` via the `flags` field of `struct io_uring_params`.

## Performance Flags

### `IORING_SETUP_SQPOLL` (5.4)
Kernel thread polls SQ for new entries. Zero syscalls for submission. The SQ poll thread sleeps after `sq_thread_idle` milliseconds of inactivity — check `IORING_SQ_NEED_WAKEUP` and call `io_uring_enter()` with `IORING_ENTER_SQ_WAKEUP` to wake it.

Requires `CAP_SYS_NICE` or setting `sq_thread_idle` (otherwise defaults to 1s). CPU can be pinned with `IORING_SETUP_SQ_AFF` + `sq_thread_cpu`.

### `IORING_SETUP_IOPOLL` (5.1)
Kernel uses polling for I/O completions instead of IRQ-driven. Only works with O_DIRECT files on supported drivers (NVMe, etc). Highest throughput for storage I/O.

### `IORING_SETUP_HYBRID_IOPOLL` (6.14)
Combines polling with sleeping — polls briefly, then sleeps. Balances latency vs CPU usage.

### `IORING_SETUP_COOP_TASKRUN` (5.19)
Task work runs cooperatively when the task transitions to/from kernel. Avoids IPIs (inter-processor interrupts). Use this if you control when you enter the kernel. Sets `IORING_SQ_TASKRUN` flag when work is pending.

### `IORING_SETUP_DEFER_TASKRUN` (6.1)
Defers task work until just before it's needed. Most aggressive task work optimization — only runs during `io_uring_enter()`. Requires `SINGLE_ISSUER`. This is what you want for event-loop designs.

### `IORING_SETUP_SINGLE_ISSUER` (6.0)
Promise that only one task submits to this ring. Enables internal optimizations. Required for `DEFER_TASKRUN`.

### `IORING_SETUP_SUBMIT_ALL` (5.19)
Don't stop submitting on error — try all SQEs. Default behavior stops at first error.

## Ring Configuration

### `IORING_SETUP_CQSIZE` (5.5)
Application specifies CQ ring size via `cq_entries`. Default is 2x SQ size. Useful when one SQE can generate multiple CQEs (multishot).

### `IORING_SETUP_CLAMP` (5.5)
Clamp ring sizes to max instead of failing. Safety net.

### `IORING_SETUP_SQE128` (5.19)
SQEs are 128 bytes instead of 64. Extra 64 bytes for passthrough commands (`URING_CMD`).

### `IORING_SETUP_CQE32` (5.19)
CQEs are 32 bytes instead of 16. Extra space for large results.

### `IORING_SETUP_CQE_MIXED` (6.14)
Allow both 16b and 32b CQEs in the same ring. 32b CQEs have `IORING_CQE_F_32` set.

### `IORING_SETUP_SQE_MIXED` (6.14)
Allow both 64b and 128b SQEs. 128b ops use dedicated opcodes (`NOP128`, `URING_CMD128`).

### `IORING_SETUP_NO_SQARRAY` (6.6)
Remove SQ indirection array. SQE index = SQ index. Less memory, less indirection.

### `IORING_SETUP_SQ_REWIND` (6.14)
Kernel always reads SQEs from index 0. No head/tail tracking. Requires `NO_SQARRAY`, incompatible with `SQPOLL`. Simplest submission model.

## Memory

### `IORING_SETUP_NO_MMAP` (6.5)
Application provides memory for rings instead of kernel allocating. Set `user_addr` in `sq_off`/`cq_off`. Useful for huge pages or pre-allocated memory.

### `IORING_SETUP_REGISTERED_FD_ONLY` (6.5)
Ring fd is registered, not a real fd. Used with `IORING_REGISTER_USE_REGISTERED_RING`. Avoids fd table lookups.

## Worker Configuration

### `IORING_SETUP_ATTACH_WQ` (5.6)
Share io-wq worker pool with another ring (specified by `wq_fd`). Reduces thread overhead for multiple rings.

### `IORING_SETUP_R_DISABLED` (5.10)
Ring starts disabled. Enable with `IORING_REGISTER_ENABLE_RINGS`. Allows setting restrictions before use.

## Recommended Setup

For a high-performance network server:

```c
struct io_uring_params p = {
    .flags = IORING_SETUP_SINGLE_ISSUER |
             IORING_SETUP_DEFER_TASKRUN |
             IORING_SETUP_COOP_TASKRUN |
             IORING_SETUP_SUBMIT_ALL |
             IORING_SETUP_NO_SQARRAY,
    .cq_entries = 4096,  // 4x SQ if using multishot
};
p.flags |= IORING_SETUP_CQSIZE;

int ring_fd = io_uring_setup(256, &p);
```

This gives you deferred task work, no SQ indirection, and generous CQ space for multishot ops.
