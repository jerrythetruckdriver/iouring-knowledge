# File Operations

io_uring supports the full POSIX file lifecycle asynchronously. No syscalls required.

## Operations Reference

### Open/Close

| Opcode | Value | Since | Equivalent |
|--------|-------|-------|------------|
| `IORING_OP_OPENAT` | 18 | 5.6 | openat(2) |
| `IORING_OP_OPENAT2` | 28 | 5.6 | openat2(2) |
| `IORING_OP_CLOSE` | 19 | 5.6 | close(2) |

```c
// Open with direct descriptor (no fd table pollution)
io_uring_prep_openat_direct(sqe, AT_FDCWD, path, O_RDONLY, 0, slot);

// Close a direct descriptor
io_uring_prep_close_direct(sqe, slot);
```

Both support `IORING_FILE_INDEX_ALLOC` for auto-allocation into the fixed file table. Since 5.15.

### Read/Write

| Opcode | Value | Since | Equivalent |
|--------|-------|-------|------------|
| `IORING_OP_READ` | 22 | 5.6 | pread(2) |
| `IORING_OP_WRITE` | 23 | 5.6 | pwrite(2) |
| `IORING_OP_READV` | 1 | 5.1 | preadv2(2) |
| `IORING_OP_WRITEV` | 2 | 5.1 | pwritev2(2) |
| `IORING_OP_READ_FIXED` | 4 | 5.1 | pread + registered buf |
| `IORING_OP_WRITE_FIXED` | 5 | 5.1 | pwrite + registered buf |
| `IORING_OP_READV_FIXED` | 58 | 6.15 | preadv + registered bufs |
| `IORING_OP_WRITEV_FIXED` | 59 | 6.15 | pwritev + registered bufs |
| `IORING_OP_READ_MULTISHOT` | 47 | 6.0 | Continuous read |

`off = -1` uses and advances file position, like read(2)/write(2).

### Metadata

| Opcode | Value | Since | Equivalent |
|--------|-------|-------|------------|
| `IORING_OP_STATX` | 21 | 5.6 | statx(2) |
| `IORING_OP_FTRUNCATE` | 53 | 6.9 | ftruncate(2) |
| `IORING_OP_FADVISE` | 24 | 5.6 | posix_fadvise(2) |
| `IORING_OP_MADVISE` | 25 | 5.6 | madvise(2) |
| `IORING_OP_FALLOCATE` | 17 | 5.6 | fallocate(2) |

### Filesystem

| Opcode | Value | Since | Equivalent |
|--------|-------|-------|------------|
| `IORING_OP_RENAMEAT` | 35 | 5.11 | renameat2(2) |
| `IORING_OP_UNLINKAT` | 36 | 5.11 | unlinkat(2) |
| `IORING_OP_MKDIRAT` | 37 | 5.15 | mkdirat(2) |
| `IORING_OP_SYMLINKAT` | 38 | 5.15 | symlinkat(2) |
| `IORING_OP_LINKAT` | 39 | 5.15 | linkat(2) |

### Extended Attributes

| Opcode | Value | Since | Equivalent |
|--------|-------|-------|------------|
| `IORING_OP_FSETXATTR` | 41 | 5.19 | fsetxattr(2) |
| `IORING_OP_SETXATTR` | 42 | 5.19 | setxattr(2) |
| `IORING_OP_FGETXATTR` | 43 | 5.19 | fgetxattr(2) |
| `IORING_OP_GETXATTR` | 44 | 5.19 | getxattr(2) |

### Sync

| Opcode | Value | Since | Equivalent |
|--------|-------|-------|------------|
| `IORING_OP_FSYNC` | 3 | 5.1 | fsync(2) |
| `IORING_OP_SYNC_FILE_RANGE` | 8 | 5.2 | sync_file_range(2) |

`IORING_FSYNC_DATASYNC` flag for fdatasync behavior.

## Patterns

### Fully Async File Copy

```c
// Linked chain: open src → open dst → read → write → close src → close dst
// With direct descriptors, zero fd table entries consumed
```

### WAL Write + Fsync (Database Pattern)

```c
// Linked pair guarantees ordering
io_uring_prep_write(sqe1, wal_fd, buf, len, offset);
sqe1->flags |= IOSQE_IO_LINK;

io_uring_prep_fsync(sqe2, wal_fd, IORING_FSYNC_DATASYNC);
```

### Prefetch with fadvise

```c
io_uring_prep_fadvise(sqe, fd, offset, len, POSIX_FADV_WILLNEED);
// Non-blocking: kernel starts readahead, you read later
```

### Metadata-Heavy Workload (ls -la equivalent)

```c
// Batch statx calls for all files in directory
for (int i = 0; i < n_files; i++) {
    io_uring_prep_statx(sqe, AT_FDCWD, paths[i], 0, STATX_BASIC_STATS, &statx_buf[i]);
}
// Single submit, all stats gathered in parallel
```

## Key Points

- File ops that need path resolution go through io-wq (they can block). Pure fd ops can complete inline.
- Direct descriptors (`file_index`) avoid fd table contention. Critical for high-throughput file serving.
- `READV_FIXED` and `WRITEV_FIXED` (6.15) combine scatter-gather with registered buffers. Best of both worlds.
- xattr ops are useful for filesystem metadata (ACLs, security labels, user attributes) without leaving the ring.
