# io_uring and SCSI Generic (sg) Devices

## Current State

No `URING_CMD` support for SCSI generic (`/dev/sg*`) devices as of kernel 6.19. The sg driver has not been updated to implement `io_uring_cmd` handlers.

## What Works

### Regular Read/Write on sg
SCSI generic uses a custom ioctl-based interface (`SG_IO`), not standard read/write. io_uring's `IORING_OP_READ`/`WRITE` don't apply to sg semantics.

### POLL_ADD
You can poll sg device fds for readiness:
```c
io_uring_prep_poll_add(sqe, sg_fd, POLLIN);
```
Tells you when an async SG_IO command has completed. But the command submission itself still requires `ioctl(SG_IO)` or `write()` with sg-specific headers.

## NVMe Passthrough: The Alternative

For NVMe devices, `URING_CMD` provides direct command passthrough (see [internals/nvme-passthrough.md](../internals/nvme-passthrough.md) and [internals/nvme-admin.md](../internals/nvme-admin.md)). This is the model for what SCSI generic support would look like.

Key differences:
- NVMe: `/dev/ng0n1` (char device) → `URING_CMD` with NVMe commands
- SCSI: `/dev/sg0` → no `URING_CMD`, must use `ioctl(SG_IO)`

## Why No SCSI URING_CMD?

1. **NVMe is the priority** — NVMe passthrough was the first `URING_CMD` use case because NVMe hardware is dominant in performance-sensitive workloads
2. **sg driver complexity** — The SCSI generic driver handles diverse hardware (HBAs, tape drives, scanners) with complex error recovery. Retrofitting `URING_CMD` is nontrivial
3. **sg v4 interface** — A new sg driver version (`sg v4`) has been in development, potentially a better place to add io_uring support
4. **Declining use case** — Direct SCSI passthrough is increasingly niche as NVMe replaces SCSI for performance workloads

## Workaround: Thread Pool

```c
// Submit SG_IO via io-wq worker thread
// Use IOSQE_ASYNC to force async execution
io_uring_prep_write(sqe, sg_fd, &sg_hdr, sizeof(sg_hdr), 0);
sqe->flags |= IOSQE_ASYNC;  // Force io-wq worker
```

This offloads the blocking `write()` (which triggers SG_IO) to an io-wq thread. Not zero-copy, not optimal, but keeps your event loop non-blocking.

## Future

If SCSI `URING_CMD` were added, it would likely follow the NVMe pattern:
- SQE `cmd_op` field for SCSI opcode
- `cmd[]` area for CDB (Command Descriptor Block)
- Fixed buffer support via `IORING_URING_CMD_FIXED`
- Possibly SQE128 for longer CDBs

For now, if you need high-performance block device access, use NVMe + `URING_CMD` passthrough instead of SCSI generic.
