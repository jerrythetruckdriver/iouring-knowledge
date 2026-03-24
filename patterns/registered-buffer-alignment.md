# Registered Buffers as DMA Targets

## Alignment Requirements

Registered buffers (`IORING_REGISTER_BUFFERS`) pin pages in physical memory. When used with O_DIRECT or NVMe passthrough, alignment matters:

### O_DIRECT Reads/Writes

- **Buffer address**: Must be aligned to `logical_sector_size` (typically 512 bytes, sometimes 4096)
- **I/O offset**: Must be aligned to `logical_sector_size`
- **I/O length**: Must be aligned to `logical_sector_size`

Query alignment:
```c
// statx for block size
struct statx stx;
statx(AT_FDCWD, "/path", 0, STATX_DIOALIGN, &stx);
// stx.stx_dio_mem_align — memory alignment
// stx.stx_dio_offset_align — offset alignment
```

### NVMe Passthrough (URING_CMD)

- **Buffer address**: Page-aligned (4096) for most NVMe controllers
- **I/O length**: Sector-aligned (512 or 4096 depending on namespace format)
- **IOPOLL**: Requires O_DIRECT, same alignment rules

### Page Boundary Rules

Registered buffers are pinned at page granularity. A buffer spanning a page boundary pins both pages. For maximum efficiency:

```c
// Page-aligned allocation for registered buffers
void *buf;
posix_memalign(&buf, 4096, buffer_size);  // or mmap with MAP_ANONYMOUS

struct iovec iov = { .iov_base = buf, .iov_len = buffer_size };
io_uring_register_buffers(&ring, &iov, 1);
```

## RLIMIT_MEMLOCK Impact

Registered buffers consume `RLIMIT_MEMLOCK` quota:

```
Pinned pages = ceil((buffer_address + buffer_size) / PAGE_SIZE) - floor(buffer_address / PAGE_SIZE)
```

An unaligned 4097-byte buffer pins 2 pages (8192 bytes of MEMLOCK quota).

**Best practice**: Page-align both address and size to avoid wasting MEMLOCK quota.

## DMA Considerations

### IOMMU/SWIOTLB

On systems with IOMMU (Intel VT-d, AMD-Vi), the kernel maps pinned pages for DMA. This is transparent — registered buffers work normally.

On systems without IOMMU using SWIOTLB (bounce buffering), registered buffers may still require a copy through the bounce buffer. io_uring can't bypass SWIOTLB.

### NUMA Locality

For NVMe devices attached to a specific NUMA node, allocate registered buffers on the same node:

```c
void *buf = numa_alloc_onnode(buffer_size, nvme_numa_node);
```

Cross-NUMA DMA works but adds latency (QPI/UPI hop).

### Huge Pages

Registered buffers backed by huge pages (2MB/1GB) reduce TLB pressure during DMA:

```c
void *buf = mmap(NULL, size, PROT_READ|PROT_WRITE,
                 MAP_PRIVATE|MAP_ANONYMOUS|MAP_HUGETLB, -1, 0);
struct iovec iov = { .iov_base = buf, .iov_len = size };
io_uring_register_buffers(&ring, &iov, 1);
```

Fewer TLB entries needed when kernel sets up DMA mappings.

## Vectored Registered Buffers (6.15)

`IORING_OP_READV_FIXED` / `IORING_OP_WRITEV_FIXED` combine vectored I/O with registered buffers:

```c
// Multiple registered buffer segments in one operation
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_readv(sqe, fd, iovs, nr_vecs, offset);
sqe->opcode = IORING_OP_READV_FIXED;
sqe->buf_index = registered_buf_index;
```

Alignment requirements apply per-iov element.

## Practical Buffer Layout

For a high-performance storage application:

```c
#define BUF_ALIGN   4096  // Page-aligned
#define BUF_SIZE    (64 * 1024)  // 64KB per buffer
#define NUM_BUFS    256

// Allocate contiguous aligned region
void *region = mmap(NULL, NUM_BUFS * BUF_SIZE,
                    PROT_READ | PROT_WRITE,
                    MAP_PRIVATE | MAP_ANONYMOUS | MAP_POPULATE,
                    -1, 0);

// Register all buffers at once
struct iovec iovs[NUM_BUFS];
for (int i = 0; i < NUM_BUFS; i++) {
    iovs[i].iov_base = (char*)region + i * BUF_SIZE;
    iovs[i].iov_len = BUF_SIZE;
}
io_uring_register_buffers(&ring, iovs, NUM_BUFS);
```

`MAP_POPULATE` pre-faults pages, avoiding page faults on first DMA. Total MEMLOCK: 256 × 64KB = 16MB.

## Anti-Patterns

1. **Unaligned small buffers**: 100-byte buffer pins full 4KB page. Waste.
2. **Scattered allocations**: malloc'd buffers may span page boundaries unpredictably
3. **Missing O_DIRECT alignment**: Buffer aligned but offset/length not — EINVAL
4. **Ignoring NUMA**: Registered buffers on wrong NUMA node add cross-node latency
5. **Not checking stx_dio_mem_align**: Assuming 512 when filesystem requires 4096
