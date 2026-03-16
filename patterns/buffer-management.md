# Buffer Management

## Three Approaches

1. **Application-managed** — you allocate, you pass pointers. Simple but no kernel-side buffer selection.
2. **Registered buffers** — pinned in kernel, referenced by index. Best for fixed I/O patterns.
3. **Provided buffer rings** — kernel picks buffers. Essential for multishot operations.

## Provided Buffer Rings (5.19+)

The modern approach. Application provides a ring of buffers; kernel selects one per completion.

### Setup

```c
struct io_uring_buf_reg reg = {
    .ring_addr = (unsigned long)buf_ring,
    .ring_entries = nr_bufs,
    .bgid = group_id,
};
io_uring_register_buf_ring(ring, &reg, 0);
```

Or let kernel allocate with `IOU_PBUF_RING_MMAP`:
```c
reg.flags = IOU_PBUF_RING_MMAP;
io_uring_register_buf_ring(ring, &reg, 0);
void *ptr = mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_SHARED,
    ring_fd, IORING_OFF_PBUF_RING | (group_id << IORING_OFF_PBUF_SHIFT));
```

### Adding Buffers

```c
io_uring_buf_ring_add(buf_ring, buf, buf_len, buf_id, mask, idx);
io_uring_buf_ring_advance(buf_ring, 1);
```

### Consuming

Kernel sets `IORING_CQE_F_BUFFER` in CQE flags. Buffer ID in upper 16 bits:
```c
int bid = cqe->flags >> IORING_CQE_BUFFER_SHIFT;
```

Return buffer to ring after processing.

## Incremental Buffer Consumption (6.10+)

With `IOU_PBUF_RING_INC`, buffers aren't fully consumed per operation. The kernel tracks consumption offset.

Register large buffers. Each recv/read consumes only what it needs. `IORING_CQE_F_BUF_MORE` indicates the buffer has more space. When cleared, buffer is fully consumed.

This is huge for reducing buffer count. One 256KB buffer serves many small reads instead of needing a separate 4KB buffer for each.

## Legacy Provided Buffers (5.7)

The old approach using `IORING_OP_PROVIDE_BUFFERS` / `IORING_OP_REMOVE_BUFFERS` opcodes. Requires SQE submission to replenish. Don't use this — use buffer rings instead.

## Registered Buffers

For fixed I/O patterns where you know which buffer goes with which operation:

```c
struct iovec iovs[N] = { ... };
io_uring_register_buffers(ring, iovs, N);

// Use
io_uring_prep_read_fixed(sqe, fd, buf, len, offset, buf_index);
```

Pages are pinned (`get_user_pages`) at registration time. Per-I/O overhead drops to near zero.

Sparse registration:
```c
struct io_uring_rsrc_register reg = {
    .nr = N,
    .flags = IORING_RSRC_REGISTER_SPARSE,
};
```

## Buffer Sizing Strategy

For network servers with multishot recv:
- **Buffer count**: 2x expected concurrent connections minimum
- **Buffer size**: Match your expected message sizes. 4KB–16KB typical for HTTP.
- **CQ size**: Large enough that you process CQEs before buffer pool exhausts

Running out of provided buffers terminates multishot. Monitor and scale.
