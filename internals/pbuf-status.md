# IORING_REGISTER_PBUF_STATUS

Inspect the state of a provided buffer group. Opcode 26.

## Problem

You registered a buffer ring, started consuming buffers, but have no idea where the kernel's head pointer is. How many buffers are available? Is the ring almost empty? Are you about to stall?

Before this: blind flying. After: you can monitor buffer pressure.

## Interface

```c
struct io_uring_buf_status {
    __u32 buf_group;    /* input: which buffer group */
    __u32 head;         /* output: kernel's head position */
    __u32 resv[8];
};
```

```c
struct io_uring_buf_status status = { .buf_group = my_bgid };
io_uring_register(ring_fd, IORING_REGISTER_PBUF_STATUS, &status, 1);
// status.head now contains the kernel's buffer consumption position
```

## What You Get

`head` is the kernel's read pointer into the buffer ring. Compare with your application's tail pointer:

```
available = (app_tail - kernel_head) & ring_mask
```

If `available` is low → replenish buffers. If zero → next recv/read with buffer select will fail with `-ENOBUFS`.

## Use Cases

### Buffer Pool Monitoring
```c
// Periodic check: are we running low?
struct io_uring_buf_status s = { .buf_group = bgid };
io_uring_register(ring_fd, IORING_REGISTER_PBUF_STATUS, &s, 1);

unsigned available = (my_tail - s.head) & ring_mask;
if (available < ring_entries / 4) {
    // Replenish: recycle processed buffers back to the ring
    refill_buffer_ring(ring, bgid, batch_size);
}
```

### Debugging Buffer Exhaustion

When multishot recv suddenly stops (CQE without `F_MORE`, res = `-ENOBUFS`):
1. Check `PBUF_STATUS` — is head == tail?
2. If yes: you're not recycling buffers fast enough
3. If no: something else terminated the multishot

### Incremental Buffer Consumption

With `IOU_PBUF_RING_INC` (6.10), buffers can be partially consumed. `PBUF_STATUS` tells you which buffer the kernel is currently reading from — essential for managing the consumption window.

## Limitations

- Synchronous register call, not an async op — incurs a syscall
- Point-in-time snapshot — head may advance between read and use
- Only tells you head position, not per-buffer consumption offset (for incremental mode, that's tracked via `CQE_F_BUF_MORE`)

## When to Use

- Production monitoring dashboards
- Adaptive buffer pool sizing
- Debugging buffer starvation in multishot patterns
- Health checks before enabling new multishot operations
