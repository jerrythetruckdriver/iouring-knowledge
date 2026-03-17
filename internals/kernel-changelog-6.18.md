# Kernel Changelog: 6.18

Released: pending (rc phase, March 2026).

## io_uring Changes

### Mixed-Size CQEs (IORING_SETUP_CQE_MIXED)

Both 16-byte and 32-byte CQEs on the same ring. `IORING_CQE_F_32` indicates a large CQE. `IORING_CQE_F_SKIP` handles gap padding at ring wrap. Reduces memory waste when only a few ops need 32-byte CQEs.

See: [internals/mixed-sqe-cqe.md](../internals/mixed-sqe-cqe.md)

### Multishot URING_CMD

`IORING_OP_URING_CMD` now supports multishot semantics. One SQE, multiple CQEs with `CQE_F_MORE`. Critical for ublk batch I/O and future driver integrations.

See: [patterns/uring-cmd-multishot.md](../patterns/uring-cmd-multishot.md)

### IORING_REGISTER_QUERY (opcode 35)

Unified capability introspection API. Query opcodes, zcrx capabilities, and sub-CQ features in one call. Replaces the scattered probe/features/trial-and-error approach.

### Zcrx Updates

Continued zero-copy receive stabilization. Multiple commits hardening the zcrx path — buffer management, refill queue, area registration.

### Request Poisoning

Debug feature that poisons freed request memory. Catches use-after-free bugs in request handling.

## Not io_uring (But Relevant)

- **PSP encryption for TCP** — Google's protocol, complements io_uring zero-copy networking
- **UDP receive scalability** — 50%+ improvement, benefits io_uring multishot recv on UDP
- **BPF signed programs** — future path to unprivileged io_uring BPF filters
- **Slub sheaves** — per-CPU slab cache, indirectly benefits io_uring's allocation-heavy paths
