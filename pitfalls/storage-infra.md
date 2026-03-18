# io_uring in Storage Infrastructure

How io_uring interacts with kernel storage layers: device mapper, encryption, and modern filesystems.

## Device Mapper (dm / LVM)

Device mapper sits between the filesystem and block device. io_uring operations on files that live on dm devices go through the standard VFS → filesystem → block layer → dm → device path. io_uring doesn't bypass or optimize the dm layer.

**dm-crypt:** Encrypted block devices add a crypto processing step. io_uring's async operations submit through dm-crypt like any other I/O. The encryption runs in dm-crypt's workqueue, not io_uring's `io-wq`. With `IOPOLL`, the poll path covers the full stack including dm-crypt completion, but the crypto processing itself isn't accelerated.

**dm-thin / dm-cache:** Thin provisioning and caching add metadata overhead. io_uring doesn't help or hurt here specifically. The metadata I/O goes through the same block layer regardless of submission mechanism.

**Key insight:** io_uring optimizes the *submission* path (fewer syscalls, batching). It doesn't optimize what happens inside the block layer. If your bottleneck is dm-crypt throughput or dm-thin metadata updates, io_uring won't help.

## NVMe Passthrough (bypassing dm)

`URING_CMD` for NVMe (`/dev/ng*`) bypasses the block layer entirely. This means:
- No dm-crypt encryption
- No dm-thin provisioning
- No filesystem
- Direct NVMe command submission

This is the io_uring power play for storage: skip everything between you and the device. But you're managing raw NVMe commands, sectors, and namespaces yourself. See [nvme-passthrough.md](../internals/nvme-passthrough.md) and [nvme-admin.md](../internals/nvme-admin.md).

## bcachefs

bcachefs (mainline since 6.7, now distributed as DKMS since 6.18) uses the standard VFS I/O path. io_uring operations on bcachefs files go through `generic_file_read_iter` / `generic_file_write_iter` like any filesystem.

bcachefs does not have io_uring-specific optimizations. Its internal I/O is submitted through the block layer's `submit_bio` path.

**Direct I/O + IOPOLL:** bcachefs supports `O_DIRECT`, so you can use io_uring's `IOPOLL` mode for polled completions. This is the most relevant io_uring feature for bcachefs: polled NVMe I/O via bcachefs avoids interrupt overhead.

## ext4 / XFS / Btrfs

All major Linux filesystems work with io_uring through the standard VFS layer:

| Feature | ext4 | XFS | Btrfs | bcachefs |
|---------|------|-----|-------|----------|
| Buffered I/O via io_uring | ✅ | ✅ | ✅ | ✅ |
| Direct I/O via io_uring | ✅ | ✅ | ✅ | ✅ |
| IOPOLL | ✅ | ✅ | ✅ | ✅ |
| fsync via io_uring | ✅ | ✅ | ✅ | ✅ |
| fallocate via io_uring | ✅ | ✅ | ✅ | ✅ |
| io_uring-specific optimizations | No | No | No | No |

XFS has historically had the best direct I/O performance and alignment handling, making it the preferred filesystem for io_uring + `IOPOLL` + NVMe workloads.

## FUSE

FUSE over io_uring (6.14) is the exception — FUSE-specific io_uring integration for kernel↔userspace filesystem communication. See [fuse-io-uring.md](../frameworks/fuse-io-uring.md).

Traditional FUSE uses `/dev/fuse` (a char device with `read`/`write` for passing requests). io_uring replaces this with direct SQ/CQ communication between the FUSE kernel module and the userspace daemon. Significant latency reduction for virtiofs, sshfs, s3fs.

## ublk

ublk uses io_uring as the entire I/O transport for userspace block devices. This is the most complete io_uring storage integration in the kernel. See [ublk.md](../frameworks/ublk.md).

## Practical Recommendations

**For databases on NVMe:**
Use io_uring with `O_DIRECT` + `IOPOLL` on XFS. Skip dm layers if you can (no encryption/LVM). This gives you the best I/O path short of NVMe passthrough.

**For encrypted storage:**
dm-crypt is the bottleneck, not the submission path. io_uring won't speed up encryption. If you need fast encrypted I/O, look at inline encryption (hardware support) or NVMe self-encrypting drives (SED).

**For userspace storage stacks:**
ublk with io_uring. This is the intended path for software-defined storage, deduplication, compression, and caching layers implemented in userspace.
