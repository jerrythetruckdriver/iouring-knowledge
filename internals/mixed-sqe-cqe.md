# Mixed-Size SQEs and CQEs

Two setup flags for heterogeneous entry sizes on a single ring.

## IORING_SETUP_CQE_MIXED (1 << 18, 6.18)

Allows both 16-byte and 32-byte CQEs on the same ring.

Before this, you had two choices:
1. `IORING_SETUP_CQE32` — all CQEs are 32 bytes (wastes space for ops that don't need it)
2. No flag — all CQEs are 16 bytes (can't get extra data from ops that produce it)

With CQE_MIXED:
- Most CQEs are 16 bytes
- CQEs that need 32 bytes set `IORING_CQE_F_32` in `cqe->flags`
- NOP with `IORING_NOP_CQE32` explicitly requests a 32-byte CQE

### CQE_F_SKIP Gap Handling

When a 32-byte CQE needs to be posted but the ring only has one 16-byte slot before wrapping, a **skip CQE** (`IORING_CQE_F_SKIP`) fills the gap. Application must ignore skip CQEs — they're padding.

```c
// Processing mixed CQEs
if (cqe->flags & IORING_CQE_F_SKIP) {
    // Padding entry, skip it
    continue;
}
if (cqe->flags & IORING_CQE_F_32) {
    // 32-byte CQE, big_cqe[] has extra data
    __u64 *extra = cqe->big_cqe;
}
```

## IORING_SETUP_SQE_MIXED (1 << 19)

Allows both 64-byte and 128-byte SQEs on the same ring.

128-byte SQEs are needed for:
- `IORING_OP_URING_CMD128` — 80 bytes of passthrough command data
- `IORING_OP_NOP128` — testing/diagnostics with 32-byte CQE

Without SQE_MIXED, you'd use `IORING_SETUP_SQE128` which makes ALL SQEs 128 bytes. That's 2x the memory for a ring where most ops are standard.

## Memory Savings

For a ring with 1024 entries where 5% of ops use URING_CMD128:

| Mode | SQ Memory | CQ Memory |
|------|-----------|-----------|
| SQE128 + CQE32 | 128 KB | 32 KB |
| SQE_MIXED + CQE_MIXED | ~68 KB | ~17 KB |

Roughly 2x savings when large entries are the minority. Which they usually are.

## Kernel Version

Both flags landed in 6.18. They're independent — you can use CQE_MIXED without SQE_MIXED and vice versa.

## Requirements

- `IORING_SETUP_NO_SQARRAY` required for SQE_MIXED
- Application must handle `CQE_F_SKIP` entries when using CQE_MIXED
