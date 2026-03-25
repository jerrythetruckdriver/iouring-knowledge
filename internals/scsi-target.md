# SCSI Target (LIO / tcmu-runner)

## Status

No URING_CMD support for SCSI target subsystem as of 6.19.

## LIO (Linux-IO Target)

LIO is the in-kernel SCSI target framework. It handles iSCSI, Fibre Channel, SRP, and vhost-scsi. The target core (`target_core_transport.c`) processes SCSI commands entirely in kernel space.

**Why no io_uring**: LIO doesn't have a userspace data path. The SCSI target engine runs in kernel context — initiator → kernel NIC → iSCSI → target core → block device. No userspace involved, so no ring to submit to.

Where LIO touches block devices (backstores), it uses the standard block layer or direct file I/O. Applications don't get to choose io_uring for this path — it's kernel-internal.

## tcmu-runner

tcmu-runner is the **userspace** SCSI target backstore handler. It implements TCMU (Target Core Module in Userspace) — the kernel sends SCSI commands to userspace via a shared memory ring (mailbox), userspace processes them, and replies.

TCMU already has its own ring-based interface:
- Shared mmap region between kernel and userspace
- Command ring (kernel → userspace)
- Response ring (userspace → kernel)
- UIO notification for new commands

**Could io_uring replace TCMU's custom ring?** Theoretically yes — this is exactly what FUSE-over-io_uring and ublk do. But:
- TCMU works fine (it's not a performance bottleneck for most targets)
- SCSI target throughput is usually limited by the network (iSCSI) or HBA
- Nobody's proposed it

## NVMe Target

Kernel NVMe target (`nvmet`) has the same architecture as LIO — entirely in-kernel. The NVMe-TCP and NVMe-RDMA transports handle I/O without userspace involvement.

No io_uring integration because there's no userspace data path to optimize.

## What Would Make Sense

If someone built a **userspace NVMe-TCP or iSCSI target** (like SPDK's target mode), they could use io_uring for:
- Socket I/O (multishot recv for protocol frames)
- Storage backend (O_DIRECT + IOPOLL for block devices)
- NVMe passthrough (URING_CMD to /dev/ng*)

SPDK already does this with its own userspace polling model (no io_uring, pure polling). The question is whether io_uring's hybrid model (poll + interrupt) could match SPDK's latency while being easier to deploy.

## Comparison

| Component | I/O Model | io_uring Potential |
|-----------|-----------|-------------------|
| LIO (kernel target) | Kernel-internal | None (no userspace) |
| tcmu-runner | Custom ring (UIO) | Could replace, no demand |
| NVMe target (kernel) | Kernel-internal | None |
| SPDK target (userspace) | Pure polling | Could complement |
| ublk (userspace block) | io_uring URING_CMD | **Already uses it** |

ublk is the proof that io_uring can serve as kernel↔userspace block I/O channel. A TCMU equivalent built on io_uring URING_CMD would work but doesn't exist yet.
