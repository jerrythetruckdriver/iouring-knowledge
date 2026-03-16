# Zero-Copy Receive (zcrx)

Added in Linux 6.15. The last major piece of the zero-copy puzzle.

## The Problem

Traditional receive: NIC → kernel buffer → `copy_to_user()` → application buffer. That copy kills throughput at 100Gbps+.

io_uring's zero-copy send (`SEND_ZC`) has existed since 6.0. Receive was harder — the kernel needs to know *where* to put data before it arrives.

## Architecture

```
NIC hardware queue
    ↓
DMA into registered memory region
    ↓
io_uring CQE with offset into region
    ↓
Application reads directly from registered memory
    ↓
Application returns buffer via Refill Queue (RQ)
```

### Key Structures

**Registration** — `io_uring_zcrx_ifq_reg`:
```c
struct io_uring_zcrx_ifq_reg {
    __u32 if_idx;       // network interface index
    __u32 if_rxq;       // hardware RX queue number
    __u32 rq_entries;   // refill queue size
    __u32 flags;
    __u64 area_ptr;     // → io_uring_zcrx_area_reg
    __u64 region_ptr;   // → io_uring_region_desc
    struct io_uring_zcrx_offsets offsets;
    __u32 zcrx_id;
    // ...
};
```

**Memory area** — `io_uring_zcrx_area_reg`:
```c
struct io_uring_zcrx_area_reg {
    __u64 addr;
    __u64 len;
    __u64 rq_area_token;
    __u32 flags;          // IORING_ZCRX_AREA_DMABUF for DMA-buf (future)
    __u32 dmabuf_fd;      // for DMA-buf backed areas
    // ...
};
```

**Completion** — `io_uring_zcrx_cqe`:
```c
struct io_uring_zcrx_cqe {
    __u64 off;    // offset into registered area
    __u64 __pad;
};
```

Offset encoding: upper bits encode area ID (`IORING_ZCRX_AREA_SHIFT = 48`), lower bits are the byte offset.

**Refill** — `io_uring_zcrx_rqe`:
```c
struct io_uring_zcrx_rqe {
    __u64 off;
    __u32 len;
    __u32 __pad;
};
```

## Flow

1. **Register**: `IORING_REGISTER_ZCRX_IFQ` with interface + RX queue + memory region
2. **Receive**: Submit `IORING_OP_RECV` with `sqe->zcrx_ifq_idx` set
3. **Complete**: CQE arrives with offset into registered memory
4. **Read**: Application reads data directly — no copy happened
5. **Return**: Push `io_uring_zcrx_rqe` to refill queue to return buffer to NIC

## Constraints

- **One NIC queue per registration**. You bind to a specific `(if_idx, if_rxq)` pair.
- **Host memory only** (6.15). DMA-buf backed areas (`IORING_ZCRX_AREA_DMABUF`) are defined in the header but not yet implemented.
- **TCP only** initially. UDP may follow.
- Requires NIC driver support for io_uring zero-copy RX path.
- Application must return buffers promptly or the NIC queue stalls.

## When to Use

- 100Gbps+ network receive
- Bulk data transfer where copy overhead is measurable
- Latency-sensitive networking where every microsecond matters

## When NOT to Use

- Most applications. If you're not saturating 25Gbps+, the copy overhead is noise.
- Small message workloads. The setup cost and complexity aren't worth it.
- Anything that needs to work on kernels < 6.15.

## vs. Zero-Copy Send

| | Send (SEND_ZC) | Receive (zcrx) |
|---|---|---|
| Kernel version | 6.0 | 6.15 |
| Mechanism | Pin user pages, DMA out | DMA into registered region |
| Notification | NOTIF CQE when send buffer reusable | Refill queue to return buffers |
| Complexity | Moderate | High — NIC queue binding, refill management |
