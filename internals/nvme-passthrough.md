# NVMe Passthrough via URING_CMD

*Kernel 5.19+ (basic), 6.x+ (mature)*

## What It Is

`IORING_OP_URING_CMD` (opcode 46) lets userspace send arbitrary commands to a file descriptor's driver. For NVMe, this means submitting NVMe commands directly through io_uring — bypassing the block layer entirely.

This is the io_uring equivalent of `ioctl(NVME_IOCTL_IO64_CMD)`, but async and batchable.

## Why It Matters

The block layer adds overhead: bio allocation, merge logic, I/O scheduler, plug/unplug. For applications that know exactly what NVMe commands they want (databases, storage engines, benchmarking tools), passthrough eliminates all of that.

With io_uring passthrough:
- **Zero block layer overhead** — commands go straight to the NVMe driver
- **Async by default** — no blocking ioctl
- **Batched submission** — multiple NVMe commands per `io_uring_enter()`
- **IOPOLL support** — polled completions for minimum latency
- **Fixed buffers** — registered buffers work with passthrough commands

## The Interface

```c
struct io_uring_sqe sqe = {
    .opcode = IORING_OP_URING_CMD,
    .fd = nvme_ns_fd,           // /dev/ngXnY (NVMe char device)
    .cmd_op = NVME_URING_CMD_IO,
    .cmd = { /* NVMe command bytes */ },
};
```

The NVMe driver defines command types:
- `NVME_URING_CMD_IO` — standard I/O command (read/write/etc.)
- `NVME_URING_CMD_IO_VEC` — vectored I/O variant
- `NVME_URING_CMD_ADMIN` — admin commands (identify, get log page, etc.)
- `NVME_URING_CMD_ADMIN_VEC` — vectored admin commands

### Character Device Required

Passthrough uses `/dev/ngXnY` (NVMe generic character device), not `/dev/nvmeXnYpZ` (block device). The character device doesn't go through the block layer.

## SQE128 for Large Commands

NVMe commands are 64 bytes. Standard SQEs are 64 bytes, with only limited space for embedded commands. Use `IORING_SETUP_SQE128` for 128-byte SQEs to fit the full NVMe command plus metadata.

With mixed-size SQE support (`IORING_SETUP_SQE_MIXED`, 6.19), you can have 64-byte SQEs for normal ops and 128-byte SQEs for passthrough on the same ring.

```c
// Ring with 128-byte SQE support
struct io_uring_params p = {
    .flags = IORING_SETUP_SQE128 | IORING_SETUP_IOPOLL,
};
```

## Fixed Buffers with Passthrough

```c
sqe.uring_cmd_flags |= IORING_URING_CMD_FIXED;
sqe.buf_index = registered_buf_idx;
```

Eliminates `get_user_pages()` on every I/O. Critical for high-IOPS passthrough workloads.

## Performance Characteristics

Passthrough + IOPOLL eliminates:
1. Block layer bio allocation and merging
2. I/O scheduler decisions
3. Interrupt handling (polled completion)
4. Per-I/O page pinning (with fixed buffers)

Real-world numbers depend heavily on device and workload, but the pattern is consistent: passthrough shaves 1-3µs per I/O compared to block layer path. At millions of IOPS, that's significant.

### When Passthrough Wins

- Random 4K I/O at queue depth > 16
- Latency-sensitive workloads (tail latency reduction)
- Custom NVMe commands (vendor-specific, ZNS zone management)
- Benchmarking raw device capability

### When Block Layer Wins

- Sequential I/O (merge logic helps)
- Shared devices (I/O scheduler fairness)
- Filesystem I/O (can't bypass block layer anyway)
- Portability (passthrough is NVMe-specific)

## Big CQE for Passthrough Results

Some NVMe commands return data in the completion. Use `IORING_SETUP_CQE32` or `IORING_SETUP_CQE_MIXED` to get 32-byte CQEs that carry the full NVMe completion queue entry.

## The 128-Byte Opcodes (6.19)

`IORING_OP_URING_CMD128` (opcode 62) is the mixed-SQE variant. On a ring with `IORING_SETUP_SQE_MIXED`, this opcode signals a 128-byte SQE while other opcodes use 64-byte SQEs.

## URING_CMD Beyond NVMe

The `URING_CMD` mechanism is generic — any driver can implement it:
- **NVMe**: I/O and admin passthrough
- **Sockets**: `io_uring_socket_op` (SIOCINQ, GETSOCKOPT, etc.)
- **FUSE**: FUSE-over-io_uring (6.14)
- **Future**: Any char device driver that implements `uring_cmd` file_operations

The pattern is: io_uring becomes a generic async command submission interface, not just an I/O ring.
