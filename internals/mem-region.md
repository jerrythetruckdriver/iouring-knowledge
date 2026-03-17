# Memory Regions (IORING_REGISTER_MEM_REGION)

*Kernel 6.13+*

## Overview

`IORING_REGISTER_MEM_REGION` (opcode 34) provides a generic mechanism for registering memory regions with the ring. It replaces ad-hoc registration for specific features (like registered wait regions) with a unified API.

## The API

```c
enum {
    IORING_MEM_REGION_TYPE_USER = 1,  // user-provided memory
};

struct io_uring_region_desc {
    __u64 user_addr;     // userspace address (if TYPE_USER)
    __u64 size;          // region size
    __u32 flags;
    __u32 id;            // assigned region ID (output)
    __u64 mmap_offset;   // for kernel-allocated regions
    __u64 __resv[4];
};

enum {
    IORING_MEM_REGION_REG_WAIT_ARG = 1,  // expose as registered wait args
};

struct io_uring_mem_region_reg {
    __u64 region_uptr;   // pointer to io_uring_region_desc
    __u64 flags;         // IORING_MEM_REGION_REG_WAIT_ARG, etc.
    __u64 __resv[2];
};
```

## Why It Exists

Before 6.13, each feature that needed shared memory with the kernel had its own registration path. Memory regions unify this:

1. **Registered wait regions** — pre-mapped wait parameters (the first consumer)
2. **Zero-copy RX** — zcrx area/refill queue registration uses `region_ptr`
3. **Future uses** — ring creation with shared huge pages, parameter passing

The pattern: register a memory region once, reference it by ID from multiple features.

## Registered Wait Integration

The primary use case in 6.13: registered wait arguments.

```c
// 1. Allocate page-aligned memory
size_t sz = sizeof(struct io_uring_reg_wait) * num_waits;
void *mem = mmap(NULL, sz, PROT_READ|PROT_WRITE,
                 MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);

// 2. Describe the region
struct io_uring_region_desc rd = {
    .user_addr = (__u64)mem,
    .size = sz,
};

// 3. Register with REG_WAIT_ARG flag
struct io_uring_mem_region_reg mr = {
    .region_uptr = (__u64)&rd,
    .flags = IORING_MEM_REGION_REG_WAIT_ARG,
};

io_uring_register(ring_fd, IORING_REGISTER_MEM_REGION, &mr, 1);

// 4. Use with IORING_ENTER_EXT_ARG_REG
// Pass index into the region instead of copying wait args every time
```

## Design Rationale

The old pattern: copy wait parameters from userspace on every `io_uring_enter()` call. With thousands of wait calls per second, that's non-trivial overhead.

The new pattern: write parameters into a pre-registered region. Pass the index. Kernel reads directly from shared memory. Zero copies for the hot path.

This is the same philosophy as registered buffers and registered files — eliminate per-operation kernel copies by pre-registering resources.

## Connection to zcrx

Zero-copy receive (`IORING_REGISTER_ZCRX_IFQ`) takes a `region_ptr` field pointing to an `io_uring_region_desc`. The refill queue lives in this registered region. Same mechanism, different consumer.

## Kernel-Allocated Regions

When `user_addr` is 0, the kernel allocates the memory and provides `mmap_offset` for the application to map it. This enables future features like shared huge page rings without the application managing the allocation.
