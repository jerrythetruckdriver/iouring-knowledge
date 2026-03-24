# io_uring_cmd for vhost-user Devices

## Status: No URING_CMD for vhost-user

vhost-user is a userspace protocol over Unix domain sockets for virtio device emulation. It uses shared memory (mmap) and eventfd for data plane, with a control channel on Unix socket.

## Architecture

```
Guest VM
  └── virtio driver
       └── vhost-user protocol (control: Unix socket, data: shared memory + eventfd)
            └── Backend process (DPDK, SPDK, custom)
```

## Where io_uring Fits

io_uring doesn't interact with vhost-user's data plane (direct memory access). But it helps with:

### Control Channel

Unix domain socket ops — setup, config, feature negotiation:

```c
// vhost-user control messages via io_uring
io_uring_prep_sendmsg(sqe, vhost_sock, &ctrl_msg, 0);  // With SCM_RIGHTS for fd passing
io_uring_prep_recvmsg(sqe, vhost_sock, &reply_msg, 0);
```

### Backend I/O

vhost-user backends (SPDK, custom) handling guest I/O requests can use io_uring for their own storage:

```c
// Backend: guest requests storage I/O → io_uring to NVMe
io_uring_prep_read(sqe, nvme_fd, buf, len, offset);
// Complete → signal guest via eventfd
io_uring_prep_write(sqe, guest_kick_fd, &val, 8, 0);
```

### eventfd Integration

vhost-user uses eventfd pairs for kick/call notifications. io_uring integrates naturally:

```c
// Wait for guest kick via io_uring
io_uring_prep_read(sqe, kick_fd, &val, 8, 0);

// Or better: POLL_ADD for edge-triggered notification
io_uring_prep_poll_add(sqe, kick_fd, POLLIN);
```

## ublk as Alternative

For block device emulation, **ublk** is the io_uring-native path:

```
Guest → virtio-blk → host kernel → ublk → io_uring → backend
```

ublk uses URING_CMD for the control/data interface. If you're building a storage backend that should be exposed as a block device, ublk is the right answer — not vhost-user.

## Comparison

| Approach | Data Plane | Control | io_uring Role |
|---|---|---|---|
| vhost-user | Shared memory + eventfd | Unix socket | Backend storage, event loop |
| ublk | URING_CMD | URING_CMD | Everything |
| virtio-blk (kernel) | QEMU block layer | QEMU | QEMU uses io_uring for block I/O |

## Bottom Line

vhost-user doesn't need URING_CMD — its data plane bypasses the kernel entirely (shared memory). io_uring helps vhost-user backends do their own I/O (storage, networking) more efficiently. For block device use cases, ublk is the io_uring-native replacement.
