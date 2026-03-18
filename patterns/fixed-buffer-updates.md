# Fixed Buffer Updates

Registered buffers are pinned in memory and mapped into the kernel's address space. Updating them at runtime requires careful management.

## Static Registration

```c
struct iovec iovs[4] = {
    { .iov_base = buf0, .iov_len = 4096 },
    { .iov_base = buf1, .iov_len = 4096 },
    { .iov_base = buf2, .iov_len = 4096 },
    { .iov_base = buf3, .iov_len = 4096 },
};
io_uring_register(ring_fd, IORING_REGISTER_BUFFERS, iovs, 4);
```

Problem: replacing buffers requires `UNREGISTER_BUFFERS` + `REGISTER_BUFFERS`. This tears down all mappings and rebuilds them. Every in-flight operation using registered buffers must complete first.

## Tagged Registration (6.0+)

```c
struct io_uring_rsrc_register reg = {
    .nr = 4,
    .data = (unsigned long)iovs,
    .tags = (unsigned long)tags,  /* non-zero tag = notify on unregister */
};
io_uring_register(ring_fd, IORING_REGISTER_BUFFERS2, &reg, sizeof(reg));
```

Tags enable notification when a buffer slot is fully unused. When you update a slot, the kernel waits for in-flight operations to finish with the old buffer, then posts a CQE with the tag value.

## Dynamic Update

```c
struct io_uring_rsrc_update2 upd = {
    .offset = 2,           /* slot index */
    .data = (unsigned long)&new_iov,
    .tags = (unsigned long)&new_tag,
    .nr = 1,
};
io_uring_register(ring_fd, IORING_REGISTER_BUFFERS_UPDATE, &upd, sizeof(upd));
```

This replaces slot 2 without touching other slots. If the old buffer had in-flight operations, they complete against the old mapping. When all references drain, a CQE with the old tag fires.

## Sparse Registration

```c
struct io_uring_rsrc_register reg = {
    .nr = 64,
    .flags = IORING_RSRC_REGISTER_SPARSE,
};
io_uring_register(ring_fd, IORING_REGISTER_BUFFERS2, &reg, sizeof(reg));
```

Creates 64 empty slots. Fill them incrementally with `REGISTER_BUFFERS_UPDATE`. Useful when you don't know all buffers upfront.

## Clone Buffers (6.13)

```c
struct io_uring_clone_buffers clone = {
    .src_fd = other_ring_fd,
    .flags = 0,
    .src_off = 0,
    .dst_off = 0,
    .nr = 16,
};
io_uring_register(ring_fd, IORING_REGISTER_CLONE_BUFFERS, &clone, sizeof(clone));
```

Share buffer tables between rings without re-pinning pages. Flags:
- `IORING_REGISTER_SRC_REGISTERED` — source fd is a registered ring fd
- `IORING_REGISTER_DST_REPLACE` — replace existing destination buffers

## Hot-Swap Pattern

For workloads that need to rotate buffers (e.g., double-buffering for video, rotating log buffers):

1. Register with tagged buffers (`REGISTER_BUFFERS2`)
2. Use buffer A for I/O operations
3. Call `REGISTER_BUFFERS_UPDATE` to swap slot to buffer B
4. Wait for tag CQE confirming buffer A is drained
5. Buffer A is now safe to reuse/free
6. Repeat

The tag CQE is your fence. Don't touch the old buffer until you get it.

## Vectored Registered Buffers (6.15)

`IORING_OP_READV_FIXED` and `IORING_OP_WRITEV_FIXED` allow scatter/gather I/O with registered buffers. Each iovec element references a registered buffer slot.

Before 6.15, registered buffers only worked with `READ_FIXED`/`WRITE_FIXED` (single buffer per operation). Vectored support enables multi-buffer I/O without giving up the pinned-page benefit.

## Memory Implications

Registered buffers are `get_user_pages()`-pinned. They cannot be swapped. Budget for this:
- 1000 × 4KB buffers = 4MB pinned
- `RLIMIT_MEMLOCK` must cover total pinned memory
- Default RLIMIT_MEMLOCK is often 64KB — you'll need to raise it

Check current pinning via `/proc/<pid>/status` → `VmPin`.
