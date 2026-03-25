# NVMe over Fabrics (NVMe-oF)

## What Is NVMe-oF

NVMe over Fabrics extends NVMe protocol over network transports: RDMA (RoCE, InfiniBand), TCP (NVMe/TCP), and Fibre Channel. Remote NVMe devices appear as local `/dev/nvmeXnY` block devices or `/dev/ngXnY` char devices.

## io_uring Integration

### Block Path (Standard)

When NVMe-oF devices are exposed as block devices (`/dev/nvmeXnY`), io_uring works transparently:

```
io_uring_prep_read(sqe, nvme_fd, buf, len, offset)
io_uring_prep_write(sqe, nvme_fd, buf, len, offset)
```

The kernel's NVMe-oF transport handles network communication. io_uring submits to the block layer; the block layer routes to the NVMe-oF driver. No difference from local NVMe at the io_uring API level.

**IOPOLL works** if the transport supports it. NVMe/TCP supports IOPOLL (polling the TCP socket), RDMA supports it natively.

### Char Device Passthrough

`/dev/ngXnY` char devices support `URING_CMD` for NVMe passthrough — including NVMe-oF targets:

```c
io_uring_prep_cmd(sqe, NVME_URING_CMD_IO, ng_fd, ...);
// Same SQE layout as local NVMe passthrough
```

**This sends NVMe commands over the fabric.** The kernel NVMe-oF host driver translates URING_CMD into fabric capsules. Fixed buffers, IOPOLL, SQE128 all work.

### Performance Characteristics

| Config | Syscalls | Latency Path |
|--------|----------|--------------|
| Block device + epoll | 2+ per I/O | syscall → block → scheduler → nvme-of → network |
| Block device + io_uring | 0 (batched) | SQE → block → scheduler → nvme-of → network |
| Block device + io_uring + IOPOLL | 0 | SQE → block → nvme-of → poll network |
| Char passthrough + URING_CMD | 0 | SQE → nvme-of direct → network |
| Char passthrough + URING_CMD + IOPOLL | 0 | SQE → nvme-of direct → poll network |

Passthrough bypasses the block layer scheduler entirely. For latency-sensitive storage (NVMe/TCP with RDMA), this matters.

## NVMe/TCP Specifics

NVMe/TCP is the most common NVMe-oF transport (no special hardware needed).

**NAPI integration:** io_uring's NAPI busy poll (`REGISTER_NAPI`, 6.9) can reduce NVMe/TCP latency by polling the network socket directly. This is the io_uring-specific advantage — epoll can't busy-poll NVMe/TCP sockets.

**Zero-copy receive (zcrx):** Theoretically, NVMe/TCP could benefit from zcrx for the data payload (skip kernel copy from NIC to application buffer). Not implemented yet — the NVMe/TCP driver manages its own receive buffers.

## NVMe-oF Target Side

The NVMe-oF **target** (server exporting storage) can use io_uring for its backend storage I/O:

```
Network → NVMe-oF target driver → io_uring → local NVMe SSD
```

SPDK does this with its own user-space NVMe-oF target. The kernel NVMe-oF target (`nvmet`) doesn't use io_uring — it submits directly to the block layer.

## SPDK Comparison

| Feature | io_uring + NVMe-oF | SPDK + NVMe-oF |
|---------|-------------------|-----------------|
| Kernel involvement | Full kernel stack | User-space only (kernel bypass) |
| Transport | Kernel NVMe/TCP, RDMA | User-space NVMe/TCP, RDMA |
| Latency | ~10-20μs (TCP), ~5-10μs (RDMA) | ~5-10μs (TCP), ~2-5μs (RDMA) |
| CPU overhead | Lower per-request (kernel optimized) | Higher baseline (polling threads) |
| Deployment | Standard kernel, no special setup | PMD drivers, hugepages, dedicated cores |

SPDK wins on raw latency. io_uring wins on operational simplicity.

## Practical Use

For NVMe-oF with io_uring, the high-value configuration is:

```
Ring: IOPOLL | SQPOLL | SINGLE_ISSUER | NO_SQARRAY
Device: /dev/ngXnY (char passthrough)
Buffers: registered, page-aligned
Transport: RDMA (if available) or NVMe/TCP
```

This gets you batched submission, polled completion, and block layer bypass over the fabric. Close to SPDK latency with zero application complexity.
