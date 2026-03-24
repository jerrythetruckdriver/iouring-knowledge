# Samba and NFS Server Implementations

## TL;DR

Neither Samba (ksmbd) nor NFS server (knfsd) uses io_uring. Both are kernel-space servers that already bypass the syscall boundary entirely.

## Why It Doesn't Apply

### knfsd (kernel NFS server)

knfsd runs in kernel threads (`nfsd`). It calls VFS functions directly — `vfs_read()`, `vfs_write()`, `vfs_fsync()`. There's no userspace→kernel transition to optimize away. io_uring's primary value proposition (batching syscalls, reducing context switches) is irrelevant when you're already in the kernel.

The NFS server's I/O path:
```
network packet → kernel NFS thread → VFS → filesystem → block layer
```

No syscalls. No io_uring needed.

### ksmbd (kernel SMB server)

Same story. ksmbd (merged in 5.15) runs as kernel threads handling SMB3 protocol in-kernel. Direct VFS calls. The data path never touches userspace.

### Samba (userspace)

Userspace Samba (`smbd`) uses synchronous I/O with its own thread pool. It could theoretically benefit from io_uring for file operations, but:

1. Samba is cross-platform (Linux, FreeBSD, macOS, Solaris)
2. The bottleneck is protocol parsing and authentication, not I/O syscalls
3. Network I/O goes through the existing event loop (tevent/epoll)
4. File I/O is already delegated to worker threads

No io_uring references in the Samba codebase.

### NFS-Ganesha (userspace NFS)

NFS-Ganesha is a userspace NFS server. Uses its own thread pool (FSAL modules) for storage backends. Could theoretically use io_uring for the storage path, but doesn't — cross-platform requirements and the storage backend abstraction layer (CephFS, GPFS, VFS, etc.) make it impractical.

## Where io_uring Could Actually Help

The one place where io_uring intersects with file servers is **FUSE over io_uring** (6.14). If your file server backend is a FUSE filesystem (sshfs, s3fs, etc.), the FUSE→io_uring optimization eliminates the `/dev/fuse` read/write cycle:

```
NFS/SMB request → VFS → FUSE → io_uring → userspace handler
```

This helps the *backend* performance, not the NFS/SMB protocol layer.

## The Pattern

Kernel-space servers don't need io_uring. They already have what io_uring provides to userspace — direct kernel function calls without syscall overhead. io_uring is a userspace optimization.

| Server | Space | io_uring | Why Not |
|--------|-------|----------|---------|
| knfsd | Kernel | No | Already in kernel |
| ksmbd | Kernel | No | Already in kernel |
| Samba | User | No | Cross-platform, thread pool |
| NFS-Ganesha | User | No | Cross-platform, FSAL abstraction |
| FUSE backends | User | Yes (6.14) | FUSE-io_uring transport |
