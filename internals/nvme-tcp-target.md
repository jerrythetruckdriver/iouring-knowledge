# NVMe-TCP Target Acceleration

## Overview

The kernel NVMe-TCP target (`nvmet-tcp`) handles NVMe commands over TCP. It runs entirely in kernel space — no userspace data path, no io_uring involvement.

But the NVMe-TCP **initiator** (client side) and **userspace targets** can benefit from io_uring.

## Kernel NVMe-TCP Target

Path: `drivers/nvme/target/tcp.c`

Architecture:
```
Initiator → TCP → nvmet-tcp → nvmet core → block device
                    ↑ kernel space, no userspace ↑
```

The target creates kernel threads that handle TCP connections. Each connection has its own receive/send work. Block I/O goes through the standard block layer to the backing device.

**No io_uring**: the entire path is kernel-internal. Adding io_uring would mean moving the target to userspace, which defeats the purpose of the kernel target.

## NVMe-TCP Initiator + io_uring

The initiator side (nvme-tcp driver, `drivers/nvme/host/tcp.c`) creates block devices. Applications access these via standard file operations or NVMe passthrough.

With io_uring:
```c
// Read from NVMe-TCP exported device via io_uring
// The application sees a regular block device
io_uring_prep_read(sqe, fd, buf, 4096, offset);
// fd is /dev/nvmeXnY (block device exposed by nvme-tcp initiator)
```

With NVMe passthrough (char device `/dev/ngXnY`):
```c
// Direct NVMe command via URING_CMD over NVMe-TCP fabric
io_uring_prep_cmd(sqe, IORING_URING_CMD_FIXED, ng_fd, ...);
// Command goes: io_uring → nvme char dev → nvme-tcp → network → target
```

This works today. The io_uring submission goes through the NVMe driver, which routes it over the fabric transparently.

## NAPI Integration

NVMe-TCP initiator benefits from NAPI busy polling (6.9+):

```c
struct io_uring_napi n = { .busy_poll_to = 50 };
io_uring_register_napi(&ring, &n);
```

This busy-polls the NIC for NVMe-TCP completions, reducing the interrupt-to-completion latency. Useful for latency-sensitive storage over TCP.

## Userspace NVMe-TCP Targets

### SPDK nvmf_tgt

SPDK's NVMe-over-Fabrics target runs in userspace with pure polling:
- DPDK for networking (no kernel TCP stack)
- Direct NVMe access via SPDK bdev
- No io_uring (doesn't need kernel involvement)

### Potential io_uring-based Target

A userspace NVMe-TCP target using io_uring would:
1. **Network**: multishot accept + multishot recv for NVMe-TCP PDUs
2. **Storage**: URING_CMD for NVMe passthrough to backing device
3. **Zero-copy**: zcrx for received data → registered buffer → NVMe write

This would be simpler to deploy than SPDK (uses kernel TCP stack) while faster than the kernel target (batched submission, registered resources, IOPOLL).

Nobody's built this yet. The kernel target is "good enough" and SPDK captures the extreme performance niche.

## Performance Hierarchy

```
Kernel NVMe-TCP target
  ↓ no io_uring, kernel-internal, simple
  ↓ ~300-400K IOPS typical

io_uring-based userspace target (hypothetical)
  ↓ io_uring for both network and storage
  ↓ registered buffers, IOPOLL, multishot
  ↓ ~500-700K IOPS estimated

SPDK nvmf_tgt
  ↓ DPDK networking, direct NVMe access
  ↓ pure polling, no kernel involvement
  ↓ ~1M+ IOPS
```

## What Exists Today

| Component | io_uring | Status |
|-----------|----------|--------|
| Kernel NVMe-TCP target | No | N/A (kernel-internal) |
| Kernel NVMe-TCP initiator | Yes | Via block/char device |
| NVMe passthrough over fabric | Yes | URING_CMD on /dev/ng* |
| NAPI for NVMe-TCP | Yes | 6.9+ |
| Userspace io_uring target | No | Doesn't exist yet |

The practical win today: use io_uring on the initiator side with NAPI busy polling for low-latency NVMe-TCP access.
