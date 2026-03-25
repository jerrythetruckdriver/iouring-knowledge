# io_uring and VFIO (GPU/Device Passthrough)

## Status: No URING_CMD for VFIO (as of 6.19)

VFIO exposes PCI devices to userspace for VM passthrough and DPDK-style device access. No URING_CMD handler exists for `/dev/vfio/*` devices.

## Why Not

**VFIO is a control-plane API:**
- `VFIO_GROUP_SET_CONTAINER`, `VFIO_SET_IOMMU`, `VFIO_DEVICE_SET_IRQS` — all ioctls
- These are one-time setup operations, not hot-path I/O
- DMA is configured once via IOMMU mappings, then the device does DMA directly

**GPU data path bypasses the kernel entirely:**
- After VFIO setup, GPU DMA reads/writes go directly to pinned memory
- No kernel involvement per-frame. io_uring can't accelerate what's already zero-overhead.
- GPU command submission goes through device-specific rings (AMD PM4, NVIDIA pushbuffers), not kernel I/O.

**Interrupt handling:**
- VFIO exposes interrupts via eventfd
- io_uring POLL_ADD on the eventfd works for async interrupt notification

## Where io_uring Intersects GPU Workloads

### DMA-BUF Zero-Copy Receive (6.16)

```
NIC → DMA-BUF → GPU memory (no CPU copy)
```

`IORING_ZCRX_AREA_DMABUF` flag on zcrx area registration. Network data lands directly in GPU-accessible memory. Use case: AI inference, video ingest, GPU-Direct networking.

### GPU → Storage Pipeline

```
GPU renders frame → io_uring WRITE to storage
// Registered buffer at GPU-accessible DMA address
// One SQE per frame, batched submission
```

### Interrupt Bridging

```c
// VFIO interrupt → eventfd → io_uring POLL_ADD
int irq_fd = eventfd(0, 0);
// Configure VFIO to signal irq_fd on device interrupt
ioctl(device_fd, VFIO_DEVICE_SET_IRQS, &irq_set);

// Monitor in io_uring event loop
io_uring_prep_poll_multishot(sqe, irq_fd, POLLIN);
```

## Comparison: Device I/O Models

| Device Type | io_uring Path | Why |
|-------------|---------------|-----|
| NVMe | URING_CMD (char dev) | NVMe has natural command model |
| Socket | Full op support | Socket ops map 1:1 to io_uring ops |
| FUSE | URING_CMD (6.14) | Kernel→userspace IPC acceleration |
| ublk | URING_CMD | Userspace block device, natural fit |
| VFIO/GPU | POLL_ADD on eventfd | Control-plane only, DMA bypasses kernel |
| USB | POLL_ADD on usbfs | URB model is already async |
| GPIO | POLL_ADD on event fd | Low bandwidth, not worth URING_CMD |

## The Pattern

VFIO and GPU devices have their own hardware-level submission/completion queues that make io_uring redundant for the data path. io_uring's role is limited to bridging device events (interrupts) into a unified event loop and handling storage/network I/O that feeds or consumes GPU data.

## See Also

- [DMA-BUF Zero-Copy Receive](dmabuf-zcrx.md) — GPU-Direct networking via io_uring
- [NVMe Passthrough](nvme-passthrough.md) — URING_CMD for a device that benefits from it
- [eventfd Integration](../patterns/eventfd-integration.md) — bridging external event sources
- [Scatter-Gather DMA](../patterns/registered-buffer-alignment.md) — buffer alignment for DMA
