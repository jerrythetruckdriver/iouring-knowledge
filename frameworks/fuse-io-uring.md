# FUSE over io_uring

Added in Linux 6.14. A case study in io_uring replacing a legacy kernel↔userspace interface.

## The Problem

FUSE (Filesystem in Userspace) uses `/dev/fuse` — a character device. The FUSE daemon reads requests from it and writes responses back. Every filesystem operation:

1. Kernel writes request to `/dev/fuse` buffer
2. Context switch to userspace FUSE daemon
3. Daemon reads from `/dev/fuse` (syscall)
4. Daemon processes request
5. Daemon writes response (syscall)
6. Context switch back to kernel
7. Kernel completes original filesystem operation

That's 2 context switches and 2 syscalls per filesystem operation. For metadata-heavy workloads (stat, readdir), this is brutal.

## The io_uring Solution

Replace the `/dev/fuse` read/write loop with io_uring ring communication:

1. FUSE daemon pre-submits receive SQEs to the ring
2. Kernel fills CQEs directly when filesystem operations arrive
3. Daemon processes request
4. Daemon submits response SQE
5. Kernel picks up response from SQ

Context switches: potentially zero (with SQPOLL or DEFER_TASKRUN).
Syscalls per operation: amortized near-zero (batching).

## Why This Matters

FUSE has been the bottleneck holding back userspace filesystems. Projects like:
- **virtiofs** (VM shared filesystems)
- **SSHFS**
- **S3-backed filesystems** (s3fs, goofys)
- **Overlay/union filesystems**

All of these pay the `/dev/fuse` penalty on every operation. io_uring communication removes the dominant overhead.

## Impact

The improvement is most visible on:
- **Metadata-heavy workloads**: `ls -la` on large directories, `find`, `stat` storms
- **Small file I/O**: Where per-operation overhead dominates transfer time
- **VM storage**: virtiofs with io_uring FUSE is significantly faster than the `/dev/fuse` path

## Broader Pattern

FUSE-over-io_uring demonstrates a general pattern: **any kernel↔userspace protocol that uses read/write on a fd can be replaced with io_uring for lower overhead**. The same approach could apply to:
- Custom device drivers
- Netlink-style interfaces
- Any request/response protocol crossing the kernel boundary

This is io_uring evolving from "async I/O API" to "general-purpose kernel↔userspace communication channel."
