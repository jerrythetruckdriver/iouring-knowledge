# READV_FIXED / WRITEV_FIXED (6.15)

## The Problem

Before 6.15, you had two choices for registered buffer I/O:

- `READ_FIXED` / `WRITE_FIXED` — single registered buffer, one contiguous range
- `READV` / `WRITEV` — scatter/gather with `iovec`, but **unregistered** buffers only

No way to do scatter/gather with registered buffers. If your data was spread across multiple registered buffer slots, you needed multiple SQEs.

## The Solution

```
IORING_OP_READV_FIXED  (opcode 58)
IORING_OP_WRITEV_FIXED (opcode 59)
```

Vectored I/O using registered buffers. Each `iovec` in the vector references a registered buffer by index.

## SQE Layout

```c
struct io_uring_sqe sqe = {
    .opcode = IORING_OP_READV_FIXED,
    .fd     = file_fd,           // or fixed file index
    .off    = file_offset,
    .addr   = (uintptr_t)iovecs, // array of struct iovec
    .len    = nr_vecs,           // number of iovecs
    .buf_index = 0,              // starting registered buffer index
};
```

Each iovec's `iov_base` must point within a registered buffer. The `buf_index` field indicates the base registered buffer index; the kernel validates that each iovec falls within the registered buffer table.

## Why It Matters

**Database engines**: WAL writes often scatter header + payload + checksum across different buffers. Previously required either:
1. Copy into a single contiguous registered buffer (defeats zero-copy)
2. Submit multiple WRITE_FIXED SQEs (more SQEs, can't be atomic)

Now: one SQE, multiple registered buffers, one write.

**Network proxies**: Forwarding data from registered recv buffers to disk — scatter/gather the received chunks directly without consolidation.

## Comparison

| Op | Registered? | Vectored? | Kernel Version |
|---|---|---|---|
| READ / WRITE | No | No | 5.6 |
| READV / WRITEV | No | Yes | 5.1 |
| READ_FIXED / WRITE_FIXED | Yes | No | 5.1 |
| READV_FIXED / WRITEV_FIXED | Yes | Yes | 6.15 |

The matrix is now complete.

## With Mixed SQE Size

READV_FIXED and WRITEV_FIXED use standard 64-byte SQEs. The iovec array is in userspace memory pointed to by `addr`, same as regular READV/WRITEV. No SQE128 needed.

## Performance Note

Registered buffers skip `get_user_pages()` on every I/O — pages are pinned at registration time. Vectored registered I/O combines this with scatter/gather, giving you the best of both worlds. The overhead vs single READ_FIXED is just the iovec array traversal.
