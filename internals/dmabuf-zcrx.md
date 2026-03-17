# DMA-BUF + io_uring Zero-Copy Receive

Kernel 6.16. The GPU/NPU zero-copy networking story.

## The Problem

Traditional zero-copy receive (zcrx, 6.15) uses kernel-allocated pages. Data arrives at the NIC, lands in kernel memory, gets mapped to userspace. Zero-copy from NIC to userspace — but if the data needs to go to a GPU or NPU, there's still a copy:

```
NIC → kernel pages → userspace mapping → GPU copy → GPU memory
                                         ^^^ this copy hurts
```

## DMA-BUF zcrx (6.16)

DMA-BUF is Linux's buffer-sharing framework for heterogeneous devices (GPUs, video codecs, NPUs, FPGAs). By using DMA-BUF backed areas for zcrx, the NIC can DMA directly into GPU-accessible memory:

```
NIC → DMA-BUF pages (GPU-accessible) → userspace mapping
                     ^^^ GPU can read directly. No copy.
```

## How It Works

### Registration

```c
struct io_uring_zcrx_area_reg area = {
    .addr = 0,          // kernel allocates
    .len = 4 * 1024 * 1024,
    .flags = IORING_ZCRX_AREA_DMABUF,
    .dmabuf_fd = gpu_dmabuf_fd,  // from GPU driver
};

struct io_uring_zcrx_ifq_reg ifq = {
    .if_idx = eth0_ifindex,
    .if_rxq = 0,
    .rq_entries = 4096,
    .area_ptr = (unsigned long)&area,
    .region_ptr = (unsigned long)&region,
};

io_uring_register(ring_fd, IORING_REGISTER_ZCRX_IFQ, &ifq, 1);
```

Key: `IORING_ZCRX_AREA_DMABUF` flag + `dmabuf_fd` from the GPU/NPU driver.

### Area ID Encoding

CQE offsets encode the area ID in the upper bits:

```c
#define IORING_ZCRX_AREA_SHIFT  48
#define IORING_ZCRX_AREA_MASK   (~(((__u64)1 << IORING_ZCRX_AREA_SHIFT) - 1))

uint16_t area_id = (cqe_off & IORING_ZCRX_AREA_MASK) >> IORING_ZCRX_AREA_SHIFT;
uint64_t offset = cqe_off & ~IORING_ZCRX_AREA_MASK;
```

### Multiple IFQs (6.16)

Kernel 6.16 also added support for multiple IFQs per ring. One ring can manage zero-copy receive from multiple NIC queues:

```
Ring → IFQ 0 (eth0 rxq0, regular pages)
     → IFQ 1 (eth0 rxq1, DMA-BUF GPU memory)
     → IFQ 2 (eth1 rxq0, DMA-BUF NPU memory)
```

## Use Cases

### 1. GPU-Direct Networking
RDMA/RoCE replacement for GPU clusters. Network data lands directly in GPU memory. No CPU involvement in the data path.

### 2. Video Ingest
Camera/encoder → NIC → DMA-BUF → GPU decode. Zero-copy end-to-end.

### 3. AI/ML Inference Pipeline
Model inputs arrive over network → land in NPU-accessible memory → inference runs → results sent back via zero-copy send.

### 4. FPGA Acceleration
Network data → DMA-BUF backed by FPGA BAR → FPGA processes in-place.

## zcrx Control Operations (6.16)

```c
enum zcrx_ctrl_op {
    ZCRX_CTRL_FLUSH_RQ,   // Flush the refill queue
    ZCRX_CTRL_EXPORT,     // Export zcrx state to another fd
};
```

`ZCRX_CTRL_EXPORT` enables cross-process zcrx sharing — one process sets up the NIC queue, another consumes packets.

## Constraints

- Requires driver support (NIC driver must implement zcrx)
- DMA-BUF fd must come from a device that supports the required DMA capabilities
- GPU memory is typically not cacheable by CPU — read-back is slow
- NIC must support header/data split or full zero-copy RX path

## The Trajectory

This is where io_uring becomes a **heterogeneous I/O fabric**:
- CPU submits intent (SQE)
- NIC delivers data directly to GPU/NPU memory (DMA-BUF zcrx)
- GPU/NPU processes in-place
- Results flow back via zero-copy send

No copies. No CPU in the data path. That's the endgame.

## Sources

- kernelnewbies.org/Linux_6.16 — DMA-BUF zcrx, multiple IFQs
- io_uring.h — `IORING_ZCRX_AREA_DMABUF`, `zcrx_ctrl_op`, `zcrx_ctrl_export`
