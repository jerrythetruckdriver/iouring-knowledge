# io_uring_cmd for USB (usbfs)

## Status: No URING_CMD for USB

As of 6.19, usbfs (`/dev/bus/usb/*`) does not implement `URING_CMD`. USB async I/O uses `ioctl(USBDEVFS_SUBMITURB)` + `reap`.

## What Works

Regular `READ`/`WRITE` on usbfs file descriptors work via io_uring, but they go through io-wq (blocking path) since usbfs doesn't support `O_DIRECT` or native async.

`POLL_ADD` on usbfs fds works for monitoring URB completion events.

## Why No URING_CMD

1. **USB URB model**: URBs (USB Request Blocks) have their own async lifecycle via `USBDEVFS_SUBMITURB` + `USBDEVFS_REAPURB`. Adding URING_CMD would duplicate existing async mechanism.
2. **Low priority**: USB isn't a hot-path I/O device. Bandwidth tops out at ~5Gbps (USB 3.1) and latency is in milliseconds.
3. **Driver complexity**: usbfs interfaces with USB core, host controller drivers, and device-specific protocols. Adding URING_CMD hooks would require changes across multiple subsystems.

## Current URING_CMD Drivers

| Driver | Device | Kernel |
|---|---|---|
| NVMe | `/dev/ng*` | 5.19 |
| Socket | TCP/UDP | 6.11 |
| FUSE | Filesystems | 6.14 |
| ublk | Block devices | 6.0 |

The pattern: URING_CMD targets high-throughput, low-latency paths where the kernel interface is the bottleneck. USB doesn't fit.

## Workaround

For high-throughput USB (video capture, SDR), use the existing URB async model with POLL_ADD to integrate into an io_uring event loop:

```c
// Submit URB via ioctl
ioctl(usbfd, USBDEVFS_SUBMITURB, &urb);

// Poll for completion via io_uring
io_uring_prep_poll_add(sqe, usbfd, POLLIN);

// On POLLIN: reap URB
ioctl(usbfd, USBDEVFS_REAPURBNDELAY, &urb_ptr);
```

This gives you io_uring-integrated USB without needing URING_CMD.
