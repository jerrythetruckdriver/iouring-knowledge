# ublk — Userspace Block Devices via io_uring

ublk is a framework for implementing block devices in userspace, with io_uring as the sole data plane transport. Added in Linux 5.19, significantly enhanced through 6.18.

## Architecture

```
Application → /dev/ublkb* → blk-mq → io_uring passthrough → ublk server → io_uring I/O
```

Every I/O to `/dev/ublkb*` is forwarded to userspace via io_uring URING_CMD. The ublk server handles the logic (qcow2 translation, network forwarding, whatever) and commits results back.

1:1 tag mapping between blk-mq requests and io_uring I/O commands. No ambiguity, no lost requests.

## Why io_uring

ublk chose io_uring passthrough commands over read/write on a char device for one reason: performance. The io_uring path avoids:
- VFS overhead per I/O
- Extra context switches
- Separate syscalls for fetch vs commit

Result: ublk achieves higher IOPS than alternatives like NBD.

## Control Plane vs Data Plane

**Control plane:** `/dev/ublk-control` misc device. ADD_DEV, START_DEV, STOP_DEV, SET_PARAMS, etc.

**Data plane:** `/dev/ublkc*` char device per ublk device. All I/O commands go through io_uring passthrough:

| Command | Purpose |
|---------|---------|
| `UBLK_U_IO_FETCH_REQ` | Register thread as I/O daemon, wait for requests |
| `UBLK_U_IO_COMMIT_AND_FETCH_REQ` | Complete current I/O + fetch next (batched) |
| `UBLK_U_IO_NEED_GET_DATA` | Deferred data copy for WRITE (compat path) |

## Zero Copy (6.15+)

ublk zero copy uses io_uring's fixed kernel buffer API:

```
UBLK_IO_REGISTER_IO_BUF   → io_buffer_register_bvec()
UBLK_IO_UNREGISTER_IO_BUF → io_buffer_unregister_bvec()
```

The block I/O request buffer is registered into the io_uring buffer table. The ublk server uses that buffer index for its own io_uring I/O — no data copy between kernel and userspace.

**Auto buffer registration** (`UBLK_F_AUTO_BUF_REG`): Kernel handles register/unregister automatically, removing I/O dependencies and enabling concurrent handling.

## Batch I/O (UBLK_F_BATCH_IO, 6.18)

Replaces per-I/O commands with per-queue batch commands:

| Command | Purpose |
|---------|---------|
| `UBLK_U_IO_PREP_IO_CMDS` | Prepare multiple I/Os in batch |
| `UBLK_U_IO_COMMIT_IO_CMDS` | Commit multiple results in batch |
| `UBLK_U_IO_FETCH_IO_CMDS` | Multishot fetch — one SQE, continuous delivery |

`FETCH_IO_CMDS` uses io_uring multishot with provided buffers. One submission fetches I/Os indefinitely. Buffer size controls batch size. Multiple tasks can submit for load balancing.

This is where ublk and multishot URING_CMD converge. ublk was one of the driving use cases for multishot URING_CMD support in 6.18.

## User Recovery

`UBLK_F_USER_RECOVERY`: Device survives server crash. `/dev/ublkb*` stays alive, new server can reconnect.

Variants:
- `UBLK_F_USER_RECOVERY_REISSUE` — re-issue in-flight I/Os (ok for read-only or idempotent writes)
- `UBLK_F_USER_RECOVERY_FAIL_IO` — fail all I/Os until recovered

## Unprivileged Mode

`UBLK_F_UNPRIVILEGED_DEV`: Container-aware. Device created in one container stays scoped to that container. Permission checks via char device path.

## The Bigger Picture

ublk validates io_uring as a general-purpose kernel↔userspace I/O channel:

- **FUSE-over-io_uring** (6.14): Same pattern for filesystems
- **ublk zero copy** (6.15): Same fixed-buffer infrastructure
- **Multishot URING_CMD** (6.18): Generalized event streaming

The trajectory: every kernel subsystem that talks to userspace will eventually use io_uring passthrough commands instead of read/write/ioctl on char devices.
