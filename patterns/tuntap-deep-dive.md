# TUN/TAP Deep Dive: URING_CMD Potential

## Current State (6.19)

TUN/TAP devices are character devices (`/dev/net/tun`) that support standard file operations. No URING_CMD handler exists.

### What Works Today

| Operation | Works | Method |
|-----------|-------|--------|
| Read packets | ✅ | `IORING_OP_READ` / `IORING_OP_READV` |
| Write packets | ✅ | `IORING_OP_WRITE` / `IORING_OP_WRITEV` |
| Poll for readability | ✅ | `IORING_OP_POLL_ADD` (multishot) |
| Multishot read | ✅ | `IORING_OP_READ_MULTISHOT` |
| Registered buffers | ✅ | Fixed buffer read/write |
| URING_CMD | ❌ | No handler in tun driver |

### Performance Pattern: Batched VPN

```c
// Setup: TUN fd for tunnel, UDP socket for wire
int tun_fd = open("/dev/net/tun", O_RDWR);
// ... ioctl(TUNSETIFF) to create interface ...

// Register both fds as fixed files
int fds[2] = { tun_fd, udp_sock };
io_uring_register_files(ring, fds, 2);

// Multishot read on TUN (packets from local stack)
sqe = io_uring_get_sqe(ring);
io_uring_prep_read_multishot(sqe, 0 /* fixed idx */, 0, 0);
sqe->flags |= IOSQE_FIXED_FILE | IOSQE_BUFFER_SELECT;
sqe->buf_group = TUN_BUF_GROUP;

// Multishot read on UDP socket (encrypted packets from wire)
sqe = io_uring_get_sqe(ring);
io_uring_prep_recv_multishot(sqe, 1 /* fixed idx */, NULL, 0, 0);
sqe->flags |= IOSQE_FIXED_FILE | IOSQE_BUFFER_SELECT;
sqe->buf_group = UDP_BUF_GROUP;
```

Event loop:
1. CQE from TUN read → encrypt packet → write to UDP socket
2. CQE from UDP recv → decrypt packet → write to TUN fd
3. Both directions batched in a single `io_uring_submit()`

## Why No URING_CMD (Yet)

URING_CMD is for device-specific commands that go beyond read/write. TUN/TAP ioctls include:

- `TUNSETIFF` — create/attach interface
- `TUNSETQUEUE` — multi-queue management
- `TUNSETVNETHDRSZ` — vnet header size
- `TUNSETSNDBUF` — send buffer size
- `TUNGETFEATURES` — query features

These are control-plane operations done once at setup. Making them async via URING_CMD has near-zero value — you call them once, not millions of times.

The data path (read/write packets) already works with standard io_uring ops. There's no hot-path ioctl that would benefit from URING_CMD.

## Comparison: TUN vs Other URING_CMD Devices

| Device | URING_CMD? | Reason |
|--------|-----------|--------|
| NVMe (/dev/ng*) | ✅ | Hot-path: I/O commands bypass block layer |
| Socket | ✅ | Hot-path: setsockopt, getsockopt, timestamps |
| FUSE | ✅ | Hot-path: filesystem requests from kernel |
| ublk | ✅ | Hot-path: block I/O commands to userspace |
| TUN/TAP | ❌ | Data path = read/write, control = rare ioctls |
| /dev/sg (SCSI) | ❌ | NVMe is the priority, SCSI declining |

The pattern is clear: URING_CMD exists for devices where the hot path requires custom commands. TUN/TAP's hot path is plain read/write.

## Multi-Queue TUN

TUN devices support multi-queue mode (`IFF_MULTI_QUEUE`). Each queue gets its own fd. This maps naturally to thread-per-core:

```
Thread 0: tun_fd_0 → Ring 0
Thread 1: tun_fd_1 → Ring 1
Thread 2: tun_fd_2 → Ring 2
```

Each thread has its own TUN queue fd, its own ring, its own provided buffers. No cross-thread contention.

## Performance vs epoll

| Metric | epoll + read/write | io_uring |
|--------|-------------------|----------|
| Syscalls per packet (bidirectional) | 4 (epoll_wait + read + write + epoll_wait) | 1 (batched submit+wait) |
| Latency | ~2-3µs per packet | ~1-2µs per packet |
| Throughput (single core) | ~500K-1M pps | ~1-2M pps |
| Memory copies | Same | Same (unless registered buffers) |

The win is syscall batching. TUN read/write goes through the same kernel path regardless of whether you use epoll or io_uring — the packet processing cost is identical.

## What Would Actually Help

If you need wire-speed TUN performance, the real bottleneck is the kernel TUN driver's per-packet processing. Alternatives:

1. **AF_XDP**: Bypass TUN entirely, process raw packets in userspace
2. **XDP redirect**: Steer packets at driver level, skip socket layer
3. **DPDK**: Full kernel bypass

io_uring makes TUN faster by reducing syscall overhead, but it can't eliminate the TUN driver's per-packet cost.
