# Mixed-Size SQEs and CQEs

## The Problem

io_uring originally forced a ring-wide choice:
- `IORING_SETUP_SQE128`: all SQEs are 128 bytes (vs default 64)
- `IORING_SETUP_CQE32`: all CQEs are 32 bytes (vs default 16)

This wastes memory when only a few operations need the larger entries (URING_CMD needs 128b SQEs for passthrough commands, NVMe needs 32b CQEs for extra completion data).

## CQE_MIXED (kernel 6.15+)

```c
#define IORING_SETUP_CQE_MIXED  (1U << 18)
```

Allows both 16-byte and 32-byte CQEs on the same ring. When a 32-byte CQE is posted, `IORING_CQE_F_32` is set in `cqe->flags`.

```c
#define IORING_CQE_F_32  (1U << 15)
```

CQ ring entry size is still 32 bytes (worst case). But normal completions use only the first 16 bytes, and the consumer knows whether to look at `big_cqe[]` by checking the flag.

**CQE_F_SKIP:** When a large CQE would wrap at the end of the ring where there's only room for one small CQE, a skip entry (`IORING_CQE_F_SKIP`) fills the gap. Application must check and skip these.

## SQE_MIXED (kernel 6.15+)

```c
#define IORING_SETUP_SQE_MIXED  (1U << 19)
```

Same idea for submissions. Both 64-byte and 128-byte SQEs on one ring. 128-byte operations use a separate opcode range (`IORING_OP_NOP128`, `IORING_OP_URING_CMD128`).

## When to Use

- NVMe passthrough (`URING_CMD`) on a ring that also does normal I/O
- Applications that occasionally need extra SQE/CQE space but don't want to double ring memory for every entry
- Databases mixing URING_CMD with standard read/write operations
