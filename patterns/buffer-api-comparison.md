# Buffer APIs: provide_buffers vs pbuf_ring

io_uring has two buffer provisioning mechanisms. The old one (`IORING_OP_PROVIDE_BUFFERS`) was the first attempt. The new one (provided buffer rings, `IORING_REGISTER_PBUF_RING`) replaced it. Use the new one.

## The Old Way: PROVIDE_BUFFERS (5.7)

```c
/* Submit an SQE to provide buffers */
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_provide_buffers(sqe, bufs, buf_size, nr_bufs, bgid, bid_start);
io_uring_submit(&ring);
/* Wait for CQE confirming registration */
```

**Problems:**
1. **It's an SQE.** Providing buffers consumes a submission slot and requires a round-trip through the CQ. Buffer refill competes with actual I/O.
2. **Kernel-side locking.** The buffer list is kernel-managed with internal locks. Under contention, this becomes a bottleneck.
3. **No incremental consumption.** Each buffer is consumed atomically. A 64KB buffer used for a 100-byte recv wastes 65436 bytes.
4. **Refill latency.** After consuming buffers, you submit more `PROVIDE_BUFFERS` SQEs. There's a window where the group is empty and operations fail with `-ENOBUFS`.

## The New Way: Provided Buffer Rings (5.19)

```c
/* Register the buffer ring */
struct io_uring_buf_reg reg = {
    .ring_addr = (unsigned long)buf_ring,
    .ring_entries = 1024,
    .bgid = 0,
    .flags = 0,
};
io_uring_register(ring_fd, IORING_REGISTER_PBUF_RING, &reg, 1);

/* Add buffers directly to the ring (userspace, no syscall) */
io_uring_buf_ring_add(br, buf, buf_size, bid, mask, offset);
io_uring_buf_ring_advance(br, 1);
```

**Why it's better:**
1. **Shared memory ring.** Buffer ring is mmap'd between kernel and userspace. No SQE needed to add buffers.
2. **Lockless refill.** Userspace writes to the ring tail, kernel reads from the head. No kernel involvement for refill.
3. **Instant availability.** Buffers are visible to the kernel as soon as you advance the tail. No round-trip.
4. **Kernel-allocated option.** `IOU_PBUF_RING_MMAP` (6.4) lets the kernel allocate the ring memory, which you mmap.

## Incremental Consumption (6.10)

```c
struct io_uring_buf_reg reg = {
    .ring_addr = (unsigned long)buf_ring,
    .ring_entries = 256,
    .bgid = 0,
    .flags = IOU_PBUF_RING_INC,  /* Enable incremental consumption */
};
```

With `IOU_PBUF_RING_INC`, a buffer isn't returned to the pool after one use. Instead:
- CQE has `IORING_CQE_F_BUF_MORE` set → buffer partially consumed, kernel keeps it
- CQE without `IORING_CQE_F_BUF_MORE` → buffer fully consumed, returned to app

This enables large buffer registration (e.g., 1MB buffers) where each recv consumes only what it needs. The kernel tracks the current offset internally.

## Introspection: PBUF_STATUS (6.4)

```c
struct io_uring_buf_status status = { .buf_group = bgid };
io_uring_register(ring_fd, IORING_REGISTER_PBUF_STATUS, &status, 1);
/* status.head = kernel's current head position */
uint32_t available = tail - status.head;
```

Use this for monitoring buffer exhaustion and debugging `-ENOBUFS` errors.

## Comparison Table

| | PROVIDE_BUFFERS | pbuf_ring |
|---|---|---|
| Kernel | 5.7+ | 5.19+ |
| Buffer add | SQE (syscall) | Userspace (no syscall) |
| Refill latency | CQ round-trip | Instant (memory write) |
| Locking | Kernel internal | Lockless ring |
| Incremental | No | Yes (6.10, `IOU_PBUF_RING_INC`) |
| Kernel alloc | No | Yes (6.4, `IOU_PBUF_RING_MMAP`) |
| Introspection | No | Yes (6.4, `REGISTER_PBUF_STATUS`) |
| Max buffers | Limited by SQ | Ring entries (power of 2) |

## Migration

If you're still using `PROVIDE_BUFFERS`:

1. Replace `io_uring_prep_provide_buffers()` with `io_uring_register_buf_ring()`
2. Replace `io_uring_prep_remove_buffers()` with `io_uring_unregister_buf_ring()`
3. Buffer add is now `io_uring_buf_ring_add()` + `io_uring_buf_ring_advance()` — no SQE needed
4. Add `IORING_CQE_F_BUF_MORE` handling if using incremental consumption

The buffer selection SQE flags (`IOSQE_BUFFER_SELECT`, `buf_group`) remain the same on the consumer side.

## Recommendation

Always use `IORING_REGISTER_PBUF_RING`. `PROVIDE_BUFFERS` exists for backward compatibility. There is no scenario where the old API wins.
