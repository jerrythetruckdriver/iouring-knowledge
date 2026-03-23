# Why There's No IORING_OP_MMAP

## The Short Answer

`mmap()` creates a virtual memory mapping. It modifies the process's page tables and VMA (Virtual Memory Area) tree. This is fundamentally a **synchronous address space operation**, not an I/O operation. io_uring does I/O.

## The Longer Answer

When you call `mmap()`:

1. Kernel allocates a VMA struct
2. Inserts it into the process's VMA tree (requires mmap_lock)
3. Updates page tables (may involve TLB shootdown across CPUs)
4. Returns a virtual address the process can immediately dereference

Steps 2-3 require **synchronous modification of the calling process's address space**. There's no meaningful way to do this asynchronously — the process needs the mapping to exist before it can use the pointer.

Even if you could submit it async:
- You'd need the virtual address back before doing anything useful
- Page faults on the mapped region are still synchronous
- The mmap_lock serializes all VMA modifications anyway

## What io_uring Does Instead

### For File I/O (replacing mmap+memcpy)

```c
// Instead of: ptr = mmap(file); memcpy(buf, ptr, len); munmap(ptr);
// Use: io_uring READ with registered buffers
sqe->opcode = IORING_OP_READ_FIXED;
sqe->fd = file_fd;
sqe->buf_index = reg_buf_idx;
sqe->len = len;
sqe->off = file_offset;
```

Registered buffers are pre-pinned and pre-mapped. READ_FIXED avoids per-operation page pinning. This is faster than mmap for sequential/random I/O patterns.

### For Shared Memory (replacing mmap MAP_SHARED)

io_uring itself uses mmap for the SQ/CQ rings. `IORING_SETUP_NO_MMAP` (6.5) lets you provide your own memory, avoiding the mmap entirely.

`IORING_REGISTER_MEM_REGION` (6.13) registers memory regions that the kernel allocates and maps for you — but this is ring setup, not a per-operation thing.

### For DMA (replacing mmap on device files)

`URING_CMD` with NVMe passthrough bypasses mmap entirely. Data goes directly between registered buffers and the device via DMA. No page table manipulation needed.

## Related Ops That Don't Exist (and Why)

| Syscall | Why no io_uring op |
|---------|-------------------|
| `mmap()` | Address space modification, not I/O |
| `munmap()` | Same — VMA tree modification |
| `mprotect()` | Page table permission change |
| `madvise()` | **IORING_OP_MADVISE exists** (5.6) — advisory, doesn't modify mappings |
| `mlock()` | Synchronous page pinning, needs completion before use |
| `brk()` | Heap management, not I/O |

`FADVISE` and `MADVISE` are the exceptions — they're hints to the kernel that don't require synchronous address space changes.

## The Pattern

io_uring replaces `mmap()`-based I/O patterns with registered buffers + direct read/write. The buffer is pinned once at setup time, used for thousands of operations, and unpinned at teardown. This is strictly better than mmap for I/O workloads:

- No per-access page faults
- No TLB misses on first access
- No mmap_lock contention
- Predictable memory usage (no lazy allocation)
- Works with O_DIRECT (mmap doesn't)

If you're using mmap to read files, switch to `READ_FIXED`. If you're using mmap for IPC, use `MSG_RING` or shared registered buffers via `CLONE_BUFFERS`.
