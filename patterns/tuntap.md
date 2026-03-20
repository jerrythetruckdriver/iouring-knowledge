# io_uring with TUN/TAP Devices

## Current State

TUN/TAP devices (`/dev/net/tun`) do **not** implement URING_CMD. There's no `uring_cmd` callback in the kernel's TUN driver.

Standard io_uring file I/O works though:

```c
// Open TUN fd via ioctl (TUNSETIFF) as usual, then:
io_uring_prep_read(sqe, tun_fd, buf, len, 0);
io_uring_prep_write(sqe, tun_fd, buf, len, 0);
```

This uses `IORING_OP_READ`/`IORING_OP_WRITE` on the TUN file descriptor. Internally, the kernel dispatches to `tun_chr_read_iter`/`tun_chr_write_iter` — the same codepath as `read()`/`write()` but without the syscall.

## What Works

| Operation | Method | Notes |
|-----------|--------|-------|
| Read packets | `IORING_OP_READ` / `IORING_OP_READV` | Works, but not multishot |
| Write packets | `IORING_OP_WRITE` / `IORING_OP_WRITEV` | Works |
| Multishot recv | ❌ | Socket op only, TUN is a char device |
| Provided buffers | Via `IORING_OP_READ` + `IOSQE_BUFFER_SELECT` | Works since 6.7 (read with buffer select) |
| Zero-copy | ❌ | No zcrx for TUN, no sendmsg_zc |
| SQPOLL | ✅ | Reduces syscalls for high packet rates |
| Registered buffers | ✅ | Pin buffers for TUN read/write |

## Patterns

### Basic Packet Processing

```c
// Submit reads in a loop, process completions
for (int i = 0; i < BATCH; i++) {
    struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
    io_uring_prep_read(sqe, tun_fd, bufs[i], MTU, 0);
    io_uring_sqe_set_data64(sqe, i);
}
io_uring_submit(&ring);
```

### VPN / Tunnel Pattern

```c
// Read from TUN → encrypt → write to socket (linked)
sqe1 = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe1, tun_fd, buf, MTU, 0);
sqe1->flags |= IOSQE_IO_LINK;

// Can't link to a dependent write (need the length).
// Use completion-driven resubmission instead.
```

Linking doesn't work here because the write length depends on the read result. Use a standard completion-driven loop: read completion → encrypt in userspace → submit write.

## Why No URING_CMD for TUN?

URING_CMD requires the driver to register a `uring_cmd` callback. The TUN driver is a character device with `read_iter`/`write_iter` — no special commands needed. The packet in/out interface is simple enough that regular I/O ops suffice.

Contrast with NVMe (complex command set) or sockets (SIOCINQ, GETSOCKOPT, etc.) where URING_CMD adds real value.

## Performance vs epoll

For a TUN device, the performance difference between io_uring and epoll is modest:

- **epoll**: `epoll_wait` → `read` → process → `write` = 3 syscalls per packet
- **io_uring**: submit batch of reads, process completions, submit writes = amortized <1 syscall per packet with SQPOLL

The win is in batching. If you're processing thousands of packets/second through a VPN tunnel, io_uring's batch submission reduces context switches. For low packet rates, it doesn't matter.

## Future

If TUN/TAP were to get URING_CMD support, the interesting operations would be:
- Multishot packet receive (one SQE, continuous packet delivery)
- Zero-copy receive via zcrx integration
- Direct buffer registration with NIC DMA

This would require significant TUN driver rework. Don't hold your breath — the simpler path is XDP for high-performance packet processing, bypassing TUN entirely.
