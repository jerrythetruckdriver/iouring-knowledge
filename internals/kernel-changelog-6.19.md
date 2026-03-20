# Kernel 6.19 — io_uring Changes

## Mixed Sized SQEs (`IORING_SETUP_SQE_MIXED`, 1<<19)

6.18 added mixed CQEs. 6.19 completes the picture with mixed SQEs.

Before: if one operation needs 128-byte SQEs (NVMe passthrough, URING_CMD128), the entire ring must use `IORING_SETUP_SQE128` — doubling memory for every SQE.

Now: set `IORING_SETUP_SQE_MIXED` and use 64-byte SQEs by default. Only the occasional 128-byte operation (identified by 128b opcodes like `IORING_OP_URING_CMD128`, `IORING_OP_NOP128`) consumes the extra space.

Requires `IORING_SETUP_NO_SQARRAY` and `IORING_SETUP_SQ_REWIND`.

Memory savings: a 1024-entry ring doing mostly networking + occasional NVMe passthrough drops from 128KB to ~66KB SQE allocation.

## REGISTER_QUERY Extensions: zcrx and Ring Layout Queries

`IORING_REGISTER_QUERY` (opcode 35, introduced 6.18) gains two new query types:

- **zcrx feature query**: returns available zero-copy receive features. Lets userspace detect `ZCRX_FEATURE_RX_PAGE_SIZE` (ability to request specific rx page size via `io_uring_zcrx_ifq_reg::rx_buf_len`).
- **SQ/CQ layout queries**: return ring size information for user-provided ring memory (`IORING_SETUP_NO_MMAP`, `IORING_MEM_REGION_TYPE_USER`). Eliminates guesswork when pre-allocating ring memory.

## `getsockname` and `getpeername` via URING_CMD

New socket URING_CMD operation: `SOCKET_URING_OP_GETSOCKNAME`. Retrieves local socket address asynchronously.

Combined with the existing `SOCKET_URING_OP_GETSOCKOPT`/`SETSOCKOPT`, the full socket URING_CMD enum is now:

```c
enum io_uring_socket_op {
    SOCKET_URING_OP_SIOCINQ      = 0,
    SOCKET_URING_OP_SIOCOUTQ,
    SOCKET_URING_OP_GETSOCKOPT,
    SOCKET_URING_OP_SETSOCKOPT,
    SOCKET_URING_OP_TX_TIMESTAMP,
    SOCKET_URING_OP_GETSOCKNAME,  // NEW in 6.19
};
```

`getpeername` is also available — both were added in the same refactoring pass.

## REGISTER_ZCRX_CTRL and RQ Flushing

New registration opcode: `IORING_REGISTER_ZCRX_CTRL` (opcode 36).

Provides out-of-band control operations for zero-copy receive:

```c
enum zcrx_ctrl_op {
    ZCRX_CTRL_FLUSH_RQ,   // flush refill queue
    ZCRX_CTRL_EXPORT,     // export zcrx ring to another fd
};
```

**FLUSH_RQ**: forces the kernel to process refill queue entries. Useful when you've returned buffers and want immediate reuse without waiting for the next recv completion.

**EXPORT**: exports a zcrx configuration to another ring via fd, enabling multi-ring zcrx sharing.

## zcrx Feature Detection

New feature flag: `ZCRX_FEATURE_RX_PAGE_SIZE` (bit 0).

When set, userspace can specify desired receive page size via `io_uring_zcrx_ifq_reg::rx_buf_len`. Without this, the kernel picks the page size.

## zcrx Registration Flags

New flag: `ZCRX_REG_IMPORT` — import an existing zcrx configuration.

## Summary

| Feature | Flag/Opcode | Impact |
|---------|-------------|--------|
| Mixed SQEs | `IORING_SETUP_SQE_MIXED` (1<<19) | Memory savings for mixed workloads |
| zcrx queries | `REGISTER_QUERY` extensions | Feature detection, memory sizing |
| getsockname | `SOCKET_URING_OP_GETSOCKNAME` | Complete async socket introspection |
| zcrx ctrl | `REGISTER_ZCRX_CTRL` (opcode 36) | Fine-grained zcrx management |
| RX page size | `ZCRX_FEATURE_RX_PAGE_SIZE` | Configurable receive buffers |

6.19 is a refinement release. No new opcodes, but fills gaps in the mixed-size ring story and matures zero-copy receive into a production-ready subsystem.
