# NVMe Admin Commands via URING_CMD

Going beyond basic passthrough — NVMe admin queue operations through io_uring.

## Two Command Queues

NVMe devices have two types of command queues:
- **I/O queues**: Read, write, compare, dataset management. These are the hot path.
- **Admin queue**: Identify, get/set features, firmware management, namespace management, format. These are control plane.

Both can be submitted through `IORING_OP_URING_CMD` on the NVMe character device (`/dev/ng*`).

## Device Nodes

```
/dev/nvme0      — controller character device (admin commands)
/dev/nvme0n1    — namespace block device (traditional block I/O)
/dev/ng0n1      — namespace character device (passthrough I/O + admin)
```

Use `/dev/ng*` for URING_CMD. The `ng` (NVMe Generic) interface supports both admin and I/O commands.

## SQE Layout for NVMe

NVMe passthrough uses 128-byte SQEs (`IORING_SETUP_SQE128`) to fit the full NVMe command:

```c
struct io_uring_sqe sqe;
sqe.opcode = IORING_OP_URING_CMD;  // or IORING_OP_URING_CMD128
sqe.fd = ng_fd;                     // /dev/ng0n1
sqe.cmd_op = NVME_URING_CMD_IO;    // or NVME_URING_CMD_ADMIN
// sqe.cmd[0..79] — NVMe command in the 80-byte extension area
```

Command types:
```c
#define NVME_URING_CMD_IO       0  // I/O queue command
#define NVME_URING_CMD_ADMIN    1  // Admin queue command
#define NVME_URING_CMD_IO_VEC   2  // Vectored I/O (since 6.x)
#define NVME_URING_CMD_ADMIN_VEC 3 // Vectored admin (since 6.x)
```

## Common Admin Commands

### Identify Controller

```c
struct nvme_passthru_cmd cmd = {
    .opcode     = 0x06,     // Identify
    .nsid       = 0,
    .addr       = (__u64)buf,
    .data_len   = 4096,
    .cdw10      = 1,        // CNS=1: Identify Controller
};
// Pack into sqe->cmd[] and submit
```

### Get Log Page (SMART/Health)

```c
struct nvme_passthru_cmd cmd = {
    .opcode     = 0x02,     // Get Log Page
    .nsid       = 0xFFFFFFFF,
    .addr       = (__u64)buf,
    .data_len   = 512,
    .cdw10      = 0x02 | ((512/4 - 1) << 16),  // LID=2 (SMART), NUMDL
};
```

### Set Features (Number of Queues)

```c
struct nvme_passthru_cmd cmd = {
    .opcode     = 0x09,     // Set Features
    .cdw10      = 0x07,     // FID=7: Number of Queues
    .cdw11      = (nq_desired - 1) | ((nq_desired - 1) << 16),
};
```

## Fixed Buffers for Passthrough

```c
sqe.uring_cmd_flags = IORING_URING_CMD_FIXED;
sqe.buf_index = registered_buf_idx;
```

Avoids `get_user_pages()` on every command. Critical for high-IOPS passthrough — registered buffers save ~1μs per op from page pinning.

## IOPOLL with NVMe

```c
// Ring setup
struct io_uring_params p = { .flags = IORING_SETUP_IOPOLL };

// Submit NVMe I/O command
sqe.opcode = IORING_OP_URING_CMD;
sqe.fd = ng_fd;  // must be opened with O_DIRECT
```

IOPOLL busy-polls the NVMe completion queue instead of waiting for interrupts. Cuts latency from ~10μs to ~3-5μs on modern NVMe. Pairs with SQPOLL for fully kernel-driven I/O.

## Mixed SQE Size (6.18)

With `IORING_SETUP_SQE_MIXED`:
- Regular ops use 64-byte SQEs
- NVMe passthrough uses 128-byte SQEs (`IORING_OP_URING_CMD128`)
- Same ring handles both

Before 6.18, you needed separate rings or `IORING_SETUP_SQE128` for all SQEs (wasting 64 bytes per non-NVMe op).

## Performance: Passthrough vs Block Layer

| Path | Stack | Latency | IOPS (single thread) |
|------|-------|---------|---------------------|
| Block layer | VFS → block → NVMe driver | ~10-15μs | ~200-400K |
| URING_CMD passthrough | io_uring → NVMe driver | ~5-8μs | ~500-800K |
| URING_CMD + IOPOLL | io_uring → NVMe driver (polled) | ~3-5μs | ~800K-1.5M |
| URING_CMD + IOPOLL + SQPOLL | Fully kernel-driven | ~2-4μs | ~1-2M |

Passthrough bypasses the block layer entirely: no I/O scheduler, no merging, no page cache. You're talking directly to the NVMe controller through io_uring. The block layer adds safety (merging, scheduling, error recovery) at the cost of latency.

## Use Cases

- **Database storage engines**: Direct NVMe access for WAL/data files, custom scheduling
- **Key-value stores**: TigerBeetle, LMDB-style direct device access
- **Storage appliances**: NVMe-oF targets, SDS controllers
- **Benchmarking**: fio with `--ioengine=io_uring_cmd` for true device-level measurement
- **Device management**: Firmware updates, health monitoring, namespace ops — all async

## Gotchas

1. **Character device required**: Must use `/dev/ng*`, not `/dev/nvme*n*` block device
2. **CAP_SYS_ADMIN or /dev permissions**: Admin commands need elevated access
3. **No page cache**: Passthrough bypasses it. Your app manages caching.
4. **Error handling**: NVMe status codes, not POSIX errno. Check CQE and NVMe completion status.
5. **128-byte SQEs**: Need `IORING_SETUP_SQE128` or `IORING_SETUP_SQE_MIXED` (6.18)
