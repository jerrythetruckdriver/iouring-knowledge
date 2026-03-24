# io_uring and Network Device Offloading

## URING_CMD for Sockets

Since kernel 6.11, `IORING_OP_URING_CMD` supports socket operations via `io_uring_socket_op`:

```c
enum io_uring_socket_op {
    SOCKET_URING_OP_SIOCINQ,        // Bytes available for read
    SOCKET_URING_OP_SIOCOUTQ,       // Bytes in send queue
    SOCKET_URING_OP_GETSOCKOPT,     // Get socket option
    SOCKET_URING_OP_SETSOCKOPT,     // Set socket option
    SOCKET_URING_OP_TX_TIMESTAMP,   // TX hardware timestamps
    SOCKET_URING_OP_GETSOCKNAME,    // Get local address (6.19)
    SOCKET_URING_OP_GETPEERNAME,    // Get peer address (6.19)
};
```

This enables async socket configuration and monitoring without syscalls.

## Hardware Offload Integration

### kTLS + io_uring

Hardware TLS offload NICs (Mellanox ConnectX-6 Dx, Intel E810) can handle TLS encryption/decryption in hardware. When combined with io_uring:

```
Application → io_uring send(data) → kTLS ULP → NIC HW encrypt → wire
Wire → NIC HW decrypt → kTLS ULP → io_uring recv CQE → Application
```

No CPU crypto overhead. See `patterns/ktls-integration.md`.

### NAPI Busy Poll + io_uring

`IORING_REGISTER_NAPI` (6.9) enables per-ring NAPI busy polling:

```
NIC DMA → Ring buffer → NAPI poll (no interrupt) → io_uring CQE
```

Eliminates interrupt→softirq→context switch overhead. See `patterns/napi-busy-poll.md`.

### Zero-Copy Receive (zcrx)

DMA-BUF backed zcrx (6.16) enables NIC-to-userspace zero-copy:

```
NIC DMA → DMA-BUF pages → zcrx area → io_uring CQE
  (no kernel copy, no page allocation)
```

See `patterns/zero-copy-rx.md` and `internals/dmabuf-zcrx.md`.

## NVMe Device Offload

NVMe passthrough via URING_CMD enables:
- Direct command submission bypassing block layer
- Hardware queue mapping (io_uring SQ → NVMe SQ)
- IOPOLL for polled completions

See `internals/nvme-passthrough.md` and `internals/nvme-admin.md`.

## What's NOT Offloadable via io_uring

### No Direct NIC Programming

io_uring doesn't program NIC hardware directly. You can't:
- Configure RSS hash tables
- Program flow director rules
- Manage NIC queues
- Configure hardware timestamping (except via URING_CMD SETSOCKOPT)

These require ethtool, netlink, or ioctl — none of which have io_uring ops.

### No XDP/eBPF via io_uring

io_uring can't attach XDP programs or configure eBPF maps. Different subsystem.

### No RDMA/InfiniBand

io_uring doesn't integrate with RDMA verbs. Different submission model entirely (ibv_post_send/ibv_post_recv with completion channels).

## The Convergence Path

```
2020: io_uring = batched syscalls
2022: + NAPI busy poll (NIC awareness)
2024: + zcrx (zero-copy from NIC)
2025: + DMA-BUF zcrx (GPU/NIC direct)
2026: + Socket URING_CMD (async NIC config)
```

Trajectory: io_uring is absorbing more NIC interaction, but it's happening at the socket/protocol level, not the device driver level. The kernel's networking stack remains the abstraction boundary.

## URING_CMD for Custom Device Offload

Writing a kernel driver with URING_CMD support:

```c
static int mydev_uring_cmd(struct io_uring_cmd *cmd, unsigned int issue_flags) {
    u16 op = cmd->sqe->uring_cmd_op;
    // Decode command, perform device operation
    // For network devices: could offload filter rules, counters, etc.
    io_uring_cmd_done(cmd, result, 0, issue_flags);
    return 0;
}

static const struct file_operations mydev_fops = {
    .uring_cmd = mydev_uring_cmd,
    // ...
};
```

See `patterns/uring-cmd-custom.md` for the full guide.
