# Scatter-Gather DMA Patterns with io_uring

## The Basics

Scatter-gather (SG) DMA lets a device read from or write to multiple non-contiguous memory regions in a single operation. io_uring supports this via vectored I/O and registered buffers.

## Vectored I/O (Software Scatter-Gather)

```c
struct iovec iovs[3] = {
    { .iov_base = header, .iov_len = 64 },
    { .iov_base = payload, .iov_len = 4096 },
    { .iov_base = trailer, .iov_len = 32 },
};

// READV/WRITEV: standard vectored I/O
io_uring_prep_readv(sqe, fd, iovs, 3, offset);
io_uring_prep_writev(sqe, fd, iovs, 3, offset);
```

The kernel coalesces these into SG lists for the block layer. With O_DIRECT, the SG list maps directly to device DMA descriptors — no intermediate copy.

## Vectored Registered Buffers (6.15)

```c
// READV_FIXED/WRITEV_FIXED: vectored I/O with pre-registered buffers
// Each iovec references a registered buffer by index
struct iovec iovs[3] = { ... };  // addresses within registered buffers
io_uring_prep_readv_fixed(sqe, fd, iovs, 3, offset, buf_index);
```

**Key advantage:** Buffer pages are already pinned and DMA-mapped at registration time. Per-I/O pin/unpin overhead eliminated.

## Alignment Requirements

For O_DIRECT + DMA:

```c
struct statx stx;
io_uring_prep_statx(sqe, fd, "", 0, STATX_DIOALIGN, &stx);
// stx.stx_dio_mem_align = memory alignment (typically 512 or 4096)
// stx.stx_dio_offset_align = offset alignment

// Allocate aligned buffers
void *buf = aligned_alloc(stx.stx_dio_mem_align, size);
```

**NVMe passthrough:** Always page-aligned (4096). The NIC DMA engine needs page boundaries.

**Registered buffers:** Each buffer in the registration array must start at a page boundary for optimal DMA mapping. Unaligned buffers waste RLIMIT_MEMLOCK quota (kernel pins whole pages).

## DMA Patterns by Workload

### Database Page I/O

```c
// Register buffer pool pages (each page-aligned)
struct iovec bufs[1024];
for (int i = 0; i < 1024; i++) {
    bufs[i].iov_base = pool + (i * 8192);  // 8KB database pages
    bufs[i].iov_len = 8192;
}
io_uring_register_buffers(ring, bufs, 1024);

// Read multiple pages with vectored registered I/O
io_uring_prep_readv_fixed(sqe, fd, page_iovs, npages, offset, 0);
```

### Network Protocol Framing

```c
// Write: header + payload + checksum (zero-copy to NIC)
struct iovec iovs[] = {
    { .iov_base = header, .iov_len = sizeof(header) },
    { .iov_base = payload, .iov_len = payload_len },
    { .iov_base = &checksum, .iov_len = 4 },
};
io_uring_prep_sendmsg_zc(sqe, sock_fd, &msg, 0);
// NIC SG DMA reads from three locations, sends as one packet
```

### NVMe Passthrough SG

```c
// NVMe read with registered buffer (device DMA directly to buffer)
io_uring_prep_cmd(sqe, IORING_URING_CMD_FIXED, nvme_fd, ...);
sqe->buf_index = registered_buf_idx;
// NVMe controller DMA-reads PRP/SGL entries → registered pages
```

## Memory Layout for Maximum DMA Efficiency

```
Page boundary ──►┌──────────────────────┐
                 │ Registered Buffer 0   │ 4KB
                 ├──────────────────────┤
                 │ Registered Buffer 1   │ 4KB
                 ├──────────────────────┤
                 │ ...                   │
                 └──────────────────────┘

Optimal: each buffer starts on a page boundary
Avoids: partial page pinning waste, IOMMU scatter for sub-page regions
```

### Huge Pages for Large Buffers

```c
// 2MB huge page = contiguous physical memory = single DMA region
void *buf = mmap(NULL, 2*1024*1024, PROT_READ|PROT_WRITE,
                 MAP_PRIVATE|MAP_ANONYMOUS|MAP_HUGETLB, -1, 0);
struct iovec iov = { .iov_base = buf, .iov_len = 2*1024*1024 };
io_uring_register_buffers(ring, &iov, 1);
// DMA controller gets one contiguous physical region, no SG needed
```

## Anti-Patterns

- **Sub-page buffers without alignment** — wastes memlock quota, forces partial page pinning
- **Many tiny registered buffers** — each registration has metadata overhead, consolidate
- **Forgetting O_DIRECT alignment** — silent fallback to buffered I/O (no error, just slow)
- **Cross-NUMA DMA** — NIC on NUMA node 0 DMA-reading buffer on NUMA node 1 = cross-interconnect traffic

## See Also

- [Registered Buffer Alignment](registered-buffer-alignment.md) — alignment deep dive
- [Huge Pages](hugepages.md) — TLB and DMA advantages
- [NVMe Passthrough](../internals/nvme-passthrough.md) — device-level scatter-gather
- [Zero-Copy Receive](zero-copy-rx.md) — zcrx DMA patterns
- [READV_FIXED/WRITEV_FIXED](../internals/readv-writev-fixed.md) — vectored registered buffer ops
