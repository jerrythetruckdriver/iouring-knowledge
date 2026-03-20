# Async Filesystem Operations

## Overview

io_uring supports most filesystem operations that traditionally require synchronous syscalls. These ops go through io-wq (worker threads) since VFS doesn't support true async for metadata operations.

## Operation Reference

### Directory Operations

```c
// mkdir
io_uring_prep_mkdirat(sqe, AT_FDCWD, "/path/to/dir", 0755);

// rmdir (via unlinkat with AT_REMOVEDIR)
io_uring_prep_unlinkat(sqe, AT_FDCWD, "/path/to/dir", AT_REMOVEDIR);

// symlink
io_uring_prep_symlinkat(sqe, "/target", AT_FDCWD, "/link");

// hardlink
io_uring_prep_linkat(sqe, AT_FDCWD, "/existing", AT_FDCWD, "/newlink", 0);
```

### File Operations

```c
// rename
io_uring_prep_renameat(sqe, AT_FDCWD, "/old", AT_FDCWD, "/new", 0);
// RENAME_NOREPLACE, RENAME_EXCHANGE supported via flags

// unlink
io_uring_prep_unlinkat(sqe, AT_FDCWD, "/file", 0);

// truncate (6.9)
io_uring_prep_ftruncate(sqe, fd, new_length);

// statx
io_uring_prep_statx(sqe, AT_FDCWD, "/path", 0, STATX_BASIC_STATS, &statxbuf);
```

### Extended Attributes

```c
// Set xattr
io_uring_prep_fsetxattr(sqe, fd, "user.key", value, value_len, 0);
io_uring_prep_setxattr(sqe, "/path", "user.key", value, value_len, 0);

// Get xattr
io_uring_prep_fgetxattr(sqe, fd, "user.key", value_buf, buf_len);
io_uring_prep_getxattr(sqe, "/path", "user.key", value_buf, buf_len);
```

### Open / Close

```c
// openat
io_uring_prep_openat(sqe, AT_FDCWD, "/path", O_RDWR, 0644);

// openat2 (with resolve flags)
struct open_how how = { .flags = O_RDWR, .resolve = RESOLVE_BENEATH };
io_uring_prep_openat2(sqe, AT_FDCWD, "/path", &how);

// close
io_uring_prep_close(sqe, fd);

// close + direct descriptor
sqe->file_index = fixed_slot; // closes a registered file slot
```

## Patterns

### Atomic File Replace

```c
// Write to temp → fsync → rename over target
// Linked chain ensures ordering

// 1. Write data
io_uring_prep_write(sqe1, tmp_fd, data, len, 0);
sqe1->flags = IOSQE_IO_LINK;

// 2. fsync the data
io_uring_prep_fsync(sqe2, tmp_fd, 0);
sqe2->flags = IOSQE_IO_LINK;

// 3. Rename over target
io_uring_prep_renameat(sqe3, AT_FDCWD, "/tmp/file.tmp",
                       AT_FDCWD, "/data/file", RENAME_NOREPLACE);
```

### Batch statx (Directory Listing)

```c
// After readdir(), stat all entries in one batch
for (int i = 0; i < n_entries; i++) {
    struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
    io_uring_prep_statx(sqe, AT_FDCWD, entries[i].name,
                        AT_SYMLINK_NOFOLLOW, STATX_BASIC_STATS,
                        &statx_bufs[i]);
    sqe->user_data = i;
}
io_uring_submit(&ring);
// Reap all N completions
```

This is where io_uring's batching helps most. `ls -l` on a directory with 10K files: 10K stat() syscalls vs 1 io_uring_enter() with 10K SQEs.

### Recursive Delete

```c
// Walk directory tree, batch unlink operations
// Process leaf files first, then directories (AT_REMOVEDIR)
// Each batch: submit N unlinkat SQEs, reap, continue
```

## Performance Notes

- **Metadata ops go through io-wq**: They're not truly async at the kernel level. io_uring offloads them to worker threads. The win is batching, not eliminating blocking.
- **statx is the big win**: Batch stat operations reduce syscall count dramatically for directory traversals.
- **rename/unlink are fast already**: Single-file operations don't benefit much from async. The benefit is in not blocking the event loop.
- **openat with direct descriptors**: Combine `IORING_FILE_INDEX_ALLOC` with openat to get a registered file descriptor directly. Avoids the regular fd table entirely.

## What's NOT Available

- `readdir` / `getdents` — no async directory listing. Must use synchronous `getdents64()`.
- `chown` / `chmod` — no dedicated op. Use the file's metadata through other means.
- `mmap` / `munmap` — fundamentally synchronous (modifies process address space).
- `mount` / `umount` — no async mount operations.
