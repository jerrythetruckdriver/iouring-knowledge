# Block Device Passthrough via URING_CMD

## Current State (6.19)

`IORING_OP_URING_CMD` supports direct passthrough for specific block device types. As of 6.19, the only block device with full URING_CMD support is **NVMe** (via `/dev/ng*` character devices).

## NVMe Passthrough

NVMe passthrough bypasses the entire block layer:

```
io_uring SQE → URING_CMD → NVMe driver → NVMe hardware
```

No filesystem, no block layer, no I/O scheduler. Direct command submission to the NVMe controller.

Device nodes:
- `/dev/nvme0n1` — block device (goes through block layer)
- `/dev/ng0n1` — NVMe generic character device (URING_CMD target)

See [nvme-passthrough.md](nvme-passthrough.md) and [nvme-admin.md](nvme-admin.md) for details.

## ublk: io_uring as Block Device Backend

ublk uses URING_CMD in the opposite direction — the *kernel* sends block I/O requests to userspace via io_uring, and userspace processes them:

```
Block I/O request → ublk driver → URING_CMD CQE → userspace daemon
userspace daemon → URING_CMD SQE (completion) → ublk driver → block I/O complete
```

See [ublk.md](../frameworks/ublk.md) for the full deep dive.

## Other Block Devices

### SCSI (sg)

No URING_CMD. `/dev/sg*` is a character device but uses a legacy ioctl interface. Workaround: `IOSQE_ASYNC` to offload blocking sg operations to io-wq. See [scsi-generic.md](../pitfalls/scsi-generic.md).

### virtio-blk

No URING_CMD from userspace. QEMU uses io_uring internally as a block I/O backend for the *host side* of virtio-blk, but guest applications interact with virtio-blk as a normal block device. See [qemu-virtio.md](../frameworks/qemu-virtio.md).

### dm (device-mapper)

dm targets (dm-linear, dm-crypt, dm-thin) sit in the block layer. io_uring requests flow through them transparently — no special interaction, no URING_CMD. See [storage-infra.md](../pitfalls/storage-infra.md).

### loop

Loop devices process I/O in kernel threads. io_uring requests go through the block layer to the loop driver, which then does file I/O on the backing file. No URING_CMD.

### zram / zswap

Kernel-internal compression, no userspace URING_CMD interface. io_uring I/O to zram goes through the normal block path.

## Why Only NVMe?

URING_CMD requires the device driver to implement the `uring_cmd` method:

```c
// In the driver's file_operations
.uring_cmd = nvme_ns_chr_uring_cmd,
```

NVMe was the first because:
1. NVMe command set maps naturally to SQE/CQE (both are submission/completion queue models)
2. NVMe passthrough has clear performance benefits (bypass block layer overhead)
3. Jens Axboe (io_uring author) was the block layer maintainer
4. The NVMe generic character device (`/dev/ng*`) was designed for this

Other block device types haven't implemented it because:
- SCSI's command model is more complex (variable-length CDBs, sense data)
- Most block devices don't benefit from bypassing the block layer
- The block layer provides merging, scheduling, and accounting that's useful

## What Could Come Next

The most likely candidates for future URING_CMD block support:

1. **SCSI passthrough** — `/dev/sg*` could get a URING_CMD interface, but the SCSI maintainers haven't shown interest
2. **Custom PCIe devices** — Any char device can implement `uring_cmd`; not limited to block
3. **More ublk-like patterns** — Kernel subsystems using io_uring as their userspace interface (FUSE already does this)

The trend is clear: io_uring is becoming the universal kernel↔userspace IPC mechanism, not just for I/O. URING_CMD is the extension point.
