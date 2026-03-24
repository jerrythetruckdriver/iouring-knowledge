# io_uring and Bluetooth HCI

## Status

No URING_CMD support for Bluetooth HCI as of 6.19.

## How Bluetooth HCI Works

Bluetooth Host Controller Interface (HCI) is accessed via:
- `/dev/hciX` — Raw HCI device (deprecated in favor of mgmt interface)
- Bluetooth socket family (`AF_BLUETOOTH`, `BTPROTO_HCI`)
- BlueZ management interface (`AF_BLUETOOTH`, `BTPROTO_L2CAP`, `BTPROTO_RFCOMM`)

## What Works with io_uring

Since Bluetooth uses sockets, standard io_uring socket operations work:

```c
// L2CAP socket
int sock = socket(AF_BLUETOOTH, SOCK_SEQPACKET, BTPROTO_L2CAP);
// ... bind, connect ...

// io_uring recv/send work
io_uring_prep_recv(sqe, sock, buf, len, 0);
io_uring_prep_send(sqe, sock, buf, len, 0);

// Multishot recv works
sqe->ioprio |= IORING_RECV_MULTISHOT;
```

RFCOMM (serial-over-Bluetooth) and SCO (audio) sockets also work with io_uring's standard socket ops.

## What Doesn't Work

- No URING_CMD for HCI commands (inquiry, scan, connect, etc.)
- No `IORING_OP_URING_CMD` for `/dev/hciX`
- No async HCI event notification via io_uring (use POLL_ADD on the socket instead)

## Why No URING_CMD

1. **Bluetooth is low-bandwidth.** Peak Bluetooth 5.x: ~2 Mbps. Syscall overhead is irrelevant at these speeds.
2. **BlueZ handles the stack.** Userspace Bluetooth is managed through BlueZ/D-Bus, not raw HCI.
3. **HCI is command-response, not I/O.** URING_CMD targets high-throughput I/O paths (NVMe, network), not control protocols.

## Practical Approach

For Bluetooth applications with io_uring:

1. Use BlueZ/D-Bus for device management (discovery, pairing)
2. Once connected, use io_uring socket ops on L2CAP/RFCOMM/SCO sockets
3. For HCI event monitoring, use `IORING_OP_POLL_ADD` (multishot) on an HCI socket

This is the right split. io_uring handles the data path; BlueZ handles the control plane.
