# Multishot URING_CMD

Added in 6.18. Extends `IORING_OP_URING_CMD` with multishot semantics.

## How It Works

A single SQE generates multiple CQEs, each with `IORING_CQE_F_MORE` set until the final one. The driver controls when completions fire — io_uring just delivers them.

This is the same pattern as multishot recv/accept, but generalized to any driver.

## SQE Setup

```c
sqe->opcode = IORING_OP_URING_CMD;
sqe->cmd_op = DRIVER_SPECIFIC_OP;
sqe->ioprio |= /* driver-specific multishot flag */;
```

The multishot behavior is driver-defined. io_uring provides the infrastructure; the driver decides when to post CQEs.

## Use Cases

### ublk Batch Fetch

ublk's `UBLK_U_IO_FETCH_IO_CMDS` uses multishot to batch-deliver I/O requests to userspace. One submission, multiple completions as requests arrive.

### NVMe Polling

A driver could use multishot URING_CMD to continuously report completed NVMe commands without re-submission.

### Socket URING_CMD

Socket operations like `SOCKET_URING_OP_TX_TIMESTAMP` could use multishot to deliver timestamps for multiple sends.

## CQE Handling

Same rules as all multishot ops:
- `CQE_F_MORE` → more completions coming
- No `CQE_F_MORE` → terminal, resubmit if needed
- Handle CQ overflow gracefully — multishot terminates on overflow

## Impact

This is a big deal. Before multishot URING_CMD, every driver notification required a new SQE submission. Now any kernel subsystem can stream events through io_uring with one SQE. It's the generalization of io_uring as an event delivery mechanism.
