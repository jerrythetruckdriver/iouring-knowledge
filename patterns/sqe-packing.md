# SQE Packing Strategies

## The Problem

A standard SQE is 64 bytes. With `IORING_SETUP_SQE128`, every SQE is 128 bytes — even if only a few operations need the extra space (NVMe passthrough, NOP128, URING_CMD128). That's 2x memory for the entire SQ ring.

Mixed SQE mode (`IORING_SETUP_SQE_MIXED`, 6.19) solves this.

## Mixed SQE Mode

```c
params.flags |= IORING_SETUP_SQE_MIXED;
```

Most SQEs use 64 bytes. When a 128-byte operation is needed, it consumes two adjacent SQ slots. The kernel detects this via the opcode — 128-byte opcodes are:
- `IORING_OP_NOP128` (opcode 61)
- `IORING_OP_URING_CMD128` (opcode 62)

### Submission Pattern

```c
// Normal 64-byte SQE
struct io_uring_sqe *sqe = io_uring_get_sqe(ring);
io_uring_prep_read(sqe, fd, buf, len, offset);

// 128-byte SQE (consumes 2 slots)
struct io_uring_sqe *sqe128 = io_uring_get_sqe(ring);
sqe128->opcode = IORING_OP_URING_CMD128;
// ... fill 128 bytes including cmd[] payload
// Note: next SQ slot is consumed, don't get_sqe() for it
```

**Critical:** After preparing a 128-byte SQE, the next SQ slot is implicitly consumed. The ring accounts for this — `io_uring_get_sqe()` handles it if liburing supports mixed mode.

## Batch Submission Optimization

Maximize SQ utilization per submit call:

### Pack Related Operations Together

```c
// Submit as a batch — one io_uring_enter() for all
io_uring_prep_read(sqe1, ...);   // 64B
io_uring_prep_read(sqe2, ...);   // 64B
io_uring_prep_write(sqe3, ...);  // 64B
io_uring_prep_fsync(sqe4, ...);  // 64B
io_uring_submit(ring);           // 1 syscall for 4 ops
```

vs. submitting one at a time (4 syscalls or 4 SQPOLL cycles).

### SQ Ring Sizing

Size the SQ ring to your maximum batch depth:

| Workload | Recommended SQ Size | Rationale |
|----------|-------------------|-----------|
| Simple server | 64–128 | Enough for accept + read/write cycle |
| Database | 256–1024 | WAL writes + prefetch + fsync batches |
| Proxy/forwarder | 512–2048 | Many concurrent connections |
| NVMe passthrough | 128–512 | High IOPS, small ring OK with IOPOLL |

Use `IORING_SETUP_CLAMP` to let the kernel cap if you overshoot.

### Avoiding SQ Waste

1. **Don't over-allocate SQ entries.** Each unused slot wastes 64 bytes (or 128 bytes in SQE128 mode).

2. **Use CQE_SKIP_SUCCESS** for fire-and-forget operations:
   ```c
   sqe->flags |= IOSQE_CQE_SKIP_SUCCESS;
   ```
   Reduces CQ pressure, lets you pack more SQEs before needing to drain completions.

3. **Bundle ops reduce SQE count.** One RECV with `IORING_RECVSEND_BUNDLE` replaces N individual recv SQEs.

4. **Multishot reduces SQE count.** One multishot accept SQE replaces infinite single-shot accepts.

5. **Link chains share the SQ ring efficiently.** A linked read→process→write uses 3 SQ slots once, not repeatedly.

## SQ_REWIND Mode (6.15)

`IORING_SETUP_SQ_REWIND` changes submission semantics:

```c
params.flags |= IORING_SETUP_NO_SQARRAY | IORING_SETUP_SQ_REWIND;
```

Instead of maintaining head/tail pointers, you always pack SQEs starting from index 0. The kernel reads them sequentially. This eliminates:
- SQ array indirection
- Head/tail pointer management
- Wraparound handling

Best for workloads that prepare a batch, submit, wait for all completions, then prepare the next batch.

## Memory Math

| Config | SQ=256 | SQ=1024 | SQ=4096 |
|--------|--------|---------|---------|
| Default (64B) | 16 KB | 64 KB | 256 KB |
| SQE128 | 32 KB | 128 KB | 512 KB |
| Mixed (90% 64B, 10% 128B) | ~18 KB | ~70 KB | ~282 KB |

Mixed mode saves significant memory when only a fraction of operations need 128-byte SQEs. For a workload that's 95% reads/writes and 5% NVMe URING_CMD, SQE_MIXED vs SQE128 saves ~45% SQ memory.

## Anti-Patterns

- **Submitting one SQE at a time without SQPOLL.** Defeats batching. Accumulate, then submit.
- **SQE128 when only a few ops need it.** Use SQE_MIXED (6.19) instead.
- **Huge SQ ring "just in case."** Start with ring resize (6.13) at a small size and grow on demand.
- **Not using CQE_SKIP_SUCCESS.** If you don't need the completion for an op, skip it. Reduces CQ pressure that indirectly creates SQ backpressure.
