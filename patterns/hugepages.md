# Huge Pages for Registered Buffers

Registered buffers pin pages and the kernel builds a bio_vec array for DMA. Huge pages (2MB/1GB) reduce TLB misses and pinning overhead.

## Why It Matters

| Aspect | 4KB pages | 2MB pages | 1GB pages |
|---|---|---|---|
| Pages for 1GB buffer | 262,144 | 512 | 1 |
| TLB entries needed | 262,144 | 512 | 1 |
| Pin time | ~ms | ~μs | ~μs |
| bio_vec entries | 262,144 | 512 | 1 |
| RLIMIT_MEMLOCK accounting | Same | Same | Same |

For NVMe passthrough with IOPOLL, reducing bio_vec entries directly reduces per-I/O overhead.

## Allocation

```c
#include <sys/mman.h>

// 2MB huge pages (transparent or explicit)
void *buf = mmap(NULL, size,
    PROT_READ | PROT_WRITE,
    MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB,
    -1, 0);

// 1GB huge pages
void *buf = mmap(NULL, size,
    PROT_READ | PROT_WRITE,
    MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB | MAP_HUGE_1GB,
    -1, 0);

// Register with io_uring
struct iovec iov = { .iov_base = buf, .iov_len = size };
io_uring_register_buffers(&ring, &iov, 1);
```

## Transparent Huge Pages (THP)

THP can silently give you 2MB pages for regular `mmap`/`malloc` allocations:

```bash
# Check THP status
cat /sys/kernel/mm/transparent_hugepage/enabled
# [always] madvise never

# Explicit THP request
madvise(buf, size, MADV_HUGEPAGE);
```

**Problem**: THP pages can be split by the kernel under memory pressure. Registered buffers pin pages, which prevents splitting — but if THP hasn't coalesced before registration, you get 4KB pages anyway.

**Recommendation**: Use explicit `MAP_HUGETLB` for registered buffers. Don't rely on THP.

## NUMA + Huge Pages

```c
#include <numaif.h>

// Allocate huge pages on specific NUMA node
void *buf = mmap(NULL, size,
    PROT_READ | PROT_WRITE,
    MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB,
    -1, 0);
unsigned long node_mask = 1UL << target_node;
mbind(buf, size, MPOL_BIND, &node_mask, max_nodes, MPOL_MF_STRICT);
```

## Huge Page Reservation

```bash
# Reserve 2MB pages at boot (add to kernel cmdline)
hugepages=1024

# Or at runtime
echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages

# 1GB pages (boot only, requires contiguous memory)
hugepagesz=1G hugepages=4

# Check availability
cat /proc/meminfo | grep -i huge
```

## When to Use

| Workload | Recommendation |
|---|---|
| NVMe passthrough + IOPOLL | 2MB pages. Fewer bio_vecs, less pinning overhead. |
| Large buffer pools (databases) | 2MB pages. TLB pressure reduction. |
| zcrx areas | Default pages fine. Kernel-allocated, size is bounded. |
| Small buffers (<64KB) | Don't bother. Overhead of huge page management exceeds benefit. |
| Provided buffer rings | 4KB pages fine. Ring metadata is small. |

## Gotchas

1. **Alignment**: `MAP_HUGETLB` returns 2MB-aligned addresses. Registration just works.
2. **RLIMIT_MEMLOCK**: Huge pages count at their full size. 1GB page = 1GB against limit.
3. **Availability**: If no huge pages reserved, `mmap` with `MAP_HUGETLB` fails with `ENOMEM`.
4. **OOM**: Pinned huge pages can't be reclaimed. Size your reservation carefully.
5. **Clone buffers**: `REGISTER_CLONE_BUFFERS` shares the pinned pages — no double pinning.
