# QEMU/KVM: io_uring in the Virtualization Stack

## Architecture Overview

io_uring appears at multiple layers in the virtualization stack:

```
┌─────────────────────────────────┐
│         Guest OS                │
│  (may use io_uring internally)  │
├─────────────────────────────────┤
│      virtio-blk / virtio-scsi   │  ← Guest-side driver
├─────────────────────────────────┤
│         QEMU                    │
│    block/io_uring.c             │  ← Host-side I/O backend
├─────────────────────────────────┤
│      Host Kernel                │
│    io_uring + NVMe/storage      │
└─────────────────────────────────┘
```

## QEMU Block I/O Backend (Host Side)

QEMU's io_uring backend handles block device I/O on the host. This is production code since QEMU 5.0.

### Source: `block/io_uring.c`

```c
typedef struct {
    Coroutine *co;
    QEMUIOVector *qiov;
    uint64_t offset;
    ssize_t ret;
    int type;
    int fd;
    BdrvRequestFlags flags;
    
    int total_read;              // For short read resubmission
    QEMUIOVector resubmit_qiov;
    CqeHandler cqe_handler;
} LuringRequest;
```

### Supported Operations

| QEMU Operation | io_uring Mapping | Notes |
|----------------|------------------|-------|
| Read | `io_uring_prep_readv` / `io_uring_prep_read` | Non-vectored for single iov (faster) |
| Write | `io_uring_prep_writev` / `io_uring_prep_writev2` | writev2 for FUA flag |
| Flush | `io_uring_prep_fsync(IORING_FSYNC_DATASYNC)` | fdatasync semantics |
| Zone Append | `io_uring_prep_writev` | Zoned storage support |

### FUA (Force Unit Access)

```c
case QEMU_AIO_WRITE:
    int luring_flags = (flags & BDRV_REQ_FUA) ? RWF_DSYNC : 0;
    if (luring_flags != 0 || qiov->niov > 1) {
        io_uring_prep_writev2(sqe, fd, qiov->iov, qiov->niov, offset, luring_flags);
    }
```

`RWF_DSYNC` via writev2 gives per-write FUA without a separate fsync. This is a real win for WAL-like workloads in VMs.

### Short Read Handling

QEMU resubmits on short reads:

```c
static void luring_resubmit_short_read(LuringRequest *req, int nread) {
    req->total_read += nread;
    remaining = req->qiov->size - req->total_read;
    qemu_iovec_concat(resubmit_qiov, req->qiov, req->total_read, remaining);
    aio_add_sqe(luring_prep_sqe, req, &req->cqe_handler);
}
```

Short reads happen with buffered I/O (page cache misses mid-read). The resubmission adjusts offset and iovec to continue from where it left off.

### Error Handling

```c
// EAGAIN retry (known SCSI issue)
if (ret == -EAGAIN || ret == -EINTR)
    resubmit();
```

EAGAIN shouldn't happen with io_uring but does occur with Linux SCSI in practice.

### Coroutine Integration

Each `LuringRequest` holds a coroutine pointer. On CQE completion, QEMU resumes the coroutine that initiated the I/O. This maps naturally to io_uring's async model — the coroutine yields at submission and resumes at completion.

## What QEMU Doesn't Use

Despite having io_uring as the block backend, QEMU doesn't use:

- **Registered files/buffers** — Each request uses regular fds and unregistered buffers
- **SQPOLL** — No polling thread
- **IOPOLL** — Could benefit NVMe passthrough scenarios
- **Linked operations** — Write+fsync are separate submissions
- **Provided buffer rings** — Fixed allocation per request
- **Multishot** — No applicable use case in block I/O

Room for optimization: registered files + registered buffers alone could save ~200ns per I/O.

## virtio-blk + io_uring: Full Stack

Guest submits block I/O → virtio-blk driver → virtqueue → QEMU processes → io_uring submission → host kernel → NVMe/storage.

If the guest *also* uses io_uring:

```
Guest io_uring → Guest kernel → virtio-blk → QEMU → Host io_uring → Host kernel → NVMe
```

That's two io_uring instances in the path. The guest's io_uring handles the guest-side batching; QEMU's io_uring handles the host-side I/O.

## virtio-net

No io_uring involvement in the networking path. virtio-net uses:
- vhost-net (kernel) — direct guest↔host networking via tap
- vhost-user (userspace) — DPDK-style shared memory

Networking doesn't go through QEMU's block layer, so QEMU's io_uring backend doesn't apply.

## vDPA (virtio Data Path Acceleration)

vDPA bypasses QEMU entirely for data plane I/O. The hardware (SmartNIC/DPU) handles virtqueue processing directly. io_uring isn't in this path.

## ublk: The io_uring-Native Alternative

For userspace block devices, [ublk](ublk.md) is the io_uring-native approach. Instead of QEMU's block layer, ublk uses `URING_CMD` for zero-copy, zero-context-switch block I/O between kernel and userspace.

ublk + io_uring could eventually replace QEMU's block backend for certain use cases (especially with ublk's zero-copy mode from 6.15 and batch I/O from 6.18).

## Sources

- QEMU `block/io_uring.c` (master, March 2026)
- QEMU `block/raw-aio.h` — AIO type definitions
- QEMU block layer documentation
