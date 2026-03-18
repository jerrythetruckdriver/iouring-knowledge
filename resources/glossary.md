# Glossary

Quick reference for io_uring terminology.

## Core Concepts

**SQ (Submission Queue)**: Ring buffer where userspace writes SQEs for the kernel to consume. Mapped into userspace via mmap.

**CQ (Completion Queue)**: Ring buffer where kernel writes CQEs for userspace to consume. Mapped into userspace via mmap.

**SQE (Submission Queue Entry)**: A 64-byte (or 128-byte) struct describing one I/O operation. Contains opcode, fd, buffer pointer, flags, user_data.

**CQE (Completion Queue Entry)**: A 16-byte (or 32-byte) struct containing the result of one operation. Contains user_data (from SQE), result code, flags.

**Ring**: The io_uring instance. Contains SQ + CQ + kernel state. Created by `io_uring_setup()`, accessed via mmap'd memory.

**user_data**: 64-bit value set in SQE, echoed back in CQE. Your tag for matching completions to requests. Not interpreted by the kernel.

## Submission

**SQPOLL**: Kernel thread (`io_sq_thread`) that polls the SQ for new entries. Eliminates `io_uring_enter()` syscalls. See [sqpoll.md](../patterns/sqpoll.md).

**IOSQE_IO_LINK**: Soft link — next SQE starts when this one completes successfully. Failure breaks the chain (subsequent linked SQEs get `-ECANCELED`).

**IOSQE_IO_HARDLINK**: Hard link — next SQE starts regardless of this one's result. Chain continues even on failure.

**IOSQE_IO_DRAIN**: Barrier. This SQE waits for all prior SQEs to complete before starting.

**IOSQE_ASYNC**: Force async execution (io-wq thread). Even if the operation could complete inline.

**IOSQE_FIXED_FILE**: The `fd` field is an index into the registered file table, not a real fd.

**IOSQE_BUFFER_SELECT**: Use provided buffers. Kernel picks a buffer from the buffer group at completion time.

**IOSQE_CQE_SKIP_SUCCESS**: Don't post a CQE if this operation succeeds. Reduces CQ traffic for fire-and-forget ops.

## Completion

**CQE_F_MORE**: This CQE is from a multishot operation. More CQEs will follow from the same SQE.

**CQE_F_BUFFER**: Upper 16 bits of `flags` contain the buffer ID (from provided buffers).

**CQE_F_BUF_MORE**: Buffer is partially consumed (incremental consumption). Same buffer ID will appear in future CQEs.

**CQE_F_NOTIF**: Notification CQE for zero-copy send. Indicates kernel is done with the buffer.

**CQE_F_SOCK_NONEMPTY**: More data available on socket after this recv.

**CQE_F_SKIP**: Padding CQE for mixed-size CQ. Ignore it.

**CQE_F_32**: This is a 32-byte CQE (in mixed CQE mode).

**CQ Overflow**: CQ is full when kernel tries to post a CQE. Kernel stalls submissions (or drops, depending on `IORING_FEAT_NODROP`). Check `IORING_SQ_CQ_OVERFLOW` flag.

## Registration

**Registered buffers**: Pre-pinned memory. Avoids `get_user_pages()` on every I/O. See [registration.md](../internals/registration.md).

**Registered files (fixed files)**: Pre-looked-up file descriptors. Avoids fd table lookup per operation. See [fixed-files.md](../patterns/fixed-files.md).

**Direct descriptors**: File descriptors that exist only in io_uring's fixed file table, never in the process fd table. Created via `IORING_FILE_INDEX_ALLOC`.

**Provided buffers**: Buffer pool managed by io_uring. Kernel selects a buffer at completion time (not submission). Enables multishot recv without pre-assigning buffers. See [buffer-management.md](../patterns/buffer-management.md).

**Provided buffer ring**: Ring-based buffer management (`IORING_REGISTER_PBUF_RING`). Replaced the old PROVIDE_BUFFERS/REMOVE_BUFFERS SQE-based approach.

**Personality**: Saved credentials. Operations can run under a different security context. See [personality.md](../patterns/personality.md).

## Advanced Features

**Multishot**: One SQE generates multiple CQEs. Used for accept, recv, poll, timeout. Indicated by `CQE_F_MORE`.

**Bundle**: Batch multiple buffers in one recv/send operation. Reduces CQ entries. `IORING_RECVSEND_BUNDLE`.

**Zero-copy send**: `IORING_OP_SEND_ZC` / `IORING_OP_SENDMSG_ZC`. Sends from user buffer without copying. Produces a notification CQE when buffer can be reused.

**Zero-copy receive (zcrx)**: `IORING_OP_RECV_ZC` (6.15). NIC writes directly to user-mapped memory. See [zero-copy-rx.md](../patterns/zero-copy-rx.md).

**URING_CMD**: Generic command submission to any kernel subsystem (NVMe, sockets, ublk, FUSE). The io_uring equivalent of ioctl, but async.

**MSG_RING**: Send a message (data or fd) from one io_uring ring to another. See [ring-messaging.md](../patterns/ring-messaging.md).

## Kernel Infrastructure

**io-wq**: io_uring's worker thread pool. Handles operations that can't complete inline or via poll. See [worker-pool.md](../internals/worker-pool.md).

**COOP_TASKRUN**: Cooperative task running. Completions are delivered when the task transitions to kernel anyway, avoiding IPIs. Recommended for single-threaded apps.

**DEFER_TASKRUN**: Even more aggressive deferral. Task work only runs when explicitly requested (via `io_uring_enter`). Requires `SINGLE_ISSUER`.

**SINGLE_ISSUER**: Only one task may submit to this ring. Enables optimizations (no locking on SQ).

## Syscalls

- `io_uring_setup(entries, params)` → Create ring, returns fd
- `io_uring_enter(fd, to_submit, min_complete, flags, ...)` → Submit and/or wait
- `io_uring_register(fd, opcode, arg, nr_args)` → Register resources

After setup, a well-configured ring (SQPOLL or batched submission) may need zero syscalls on the data path.
