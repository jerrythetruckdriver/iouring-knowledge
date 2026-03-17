# Kernel Changelog: io_uring in 6.15–6.17

## Linux 6.15 (May 2025)

The zero-copy release.

### Zero-Copy Receive (zcrx)
Receive network data directly into application memory. No kernel→userspace copy. The headline feature.

- Zero-copy rx into userspace pages, eliminating the copy
- Only host memory in this version; DMA-BUF and other memory types planned
- Uses `IORING_REGISTER_ZCRX_IFQ` for interface queue setup

### IORING_OP_EPOLL_WAIT
Read epoll events via io_uring. Sounds backwards, but it's a migration bridge: existing epoll event loops can partially convert to completion-based I/O while keeping readiness-based fallbacks for things that don't have io_uring ops yet.

### Vectored Registered Buffers
Scatter/gather I/O with registered buffers. Previously, registered buffers were single contiguous regions. Now you can use `IORING_OP_READV_FIXED` / `IORING_OP_WRITEV_FIXED` with registered buffer vectors.

Used by ublk/stripe for kernel bvec buffer support.

### Other
- Toggle iowait usage when waiting on CQEs
- ublk zero-copy support (leveraging io_uring infrastructure)

---

## Linux 6.16 (July 2025)

Incremental but important.

### Multiple IFQs per Ring
Multiple zero-copy receive interface queues on a single io_uring instance. Needed for servers handling traffic on multiple NICs or NIC queues. One ring to rule them all.

### DMA-BUF Support for zcrx
Zero-copy receive directly into DMA-BUF memory. This is the GPU/NPU path — network data lands in device-accessible memory without touching the CPU. Uses `IORING_ZCRX_AREA_DMABUF` flag in `io_uring_zcrx_area_reg`.

```c
struct io_uring_zcrx_area_reg {
    __u64 addr;
    __u64 len;
    __u64 rq_area_token;
    __u32 flags;          // IORING_ZCRX_AREA_DMABUF
    __u32 dmabuf_fd;      // DMA-BUF file descriptor
    __u64 __resv2[2];
};
```

### IORING_OP_PIPE
Create pipes asynchronously via io_uring. Opcode 60.

### zcrx Control Operations
New `zcrx_ctrl` struct for flushing refill queues and exporting zcrx state:
- `ZCRX_CTRL_FLUSH_RQ` — flush the refill queue
- `ZCRX_CTRL_EXPORT` — export zcrx state to another fd

---

## Linux 6.17 (September 2025)

Quiet on the io_uring front. No major new ops. The focus was elsewhere (CPU bug mitigations, file_getattr/file_setattr syscalls, priority inheritance).

io_uring changes were internal cleanup and stabilization of the 6.15-6.16 features.

---

## Version Feature Matrix (6.15–6.17)

| Feature | Kernel |
|---------|--------|
| Zero-copy receive (zcrx) | 6.15 |
| IORING_OP_EPOLL_WAIT | 6.15 |
| IORING_OP_READV_FIXED / WRITEV_FIXED | 6.15 |
| Vectored registered buffers | 6.15 |
| ublk zero-copy | 6.15 |
| Multiple zcrx IFQs per ring | 6.16 |
| DMA-BUF zcrx | 6.16 |
| IORING_OP_PIPE | 6.16 |
| zcrx control ops (flush, export) | 6.16 |

## What's Coming

The header (as of March 2026) shows:
- `IORING_REGISTER_BPF_FILTER` (opcode 37) — BPF-based request filtering
- `IORING_REGISTER_QUERY` (opcode 35) — unified capability query
- `ZCRX_FEATURE_RX_PAGE_SIZE` — configurable zcrx page sizes

The zero-copy story is now feature-complete for the common case. The next frontier is DMA-BUF integration for AI/ML workloads where network→GPU transfers need zero CPU involvement.
