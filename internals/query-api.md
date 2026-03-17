# IORING_REGISTER_QUERY (opcode 35)

## What

A unified query interface for introspecting io_uring capabilities at runtime. Added in kernel 6.19.

```c
IORING_REGISTER_QUERY = 35,
```

## Why

Before REGISTER_QUERY, capability detection was scattered:
- **Opcode support:** `IORING_REGISTER_PROBE` (opcode 8) — tells you which opcodes exist
- **Feature flags:** `io_uring_params.features` after `io_uring_setup()` — tells you kernel-side features
- **Everything else:** Trial and error. Want to know if SQPOLL works with your config? Try it.

REGISTER_QUERY consolidates this. Instead of probing individual capabilities through different mechanisms, you get a structured query interface.

## How

The query API is defined in a separate header: `linux/io_uring/query.h`

Query categories include:
- Opcode availability (superset of REGISTER_PROBE)
- Registration operation availability
- Feature flag interrogation
- Per-opcode capability details (which flags does this opcode support?)

## vs REGISTER_PROBE

REGISTER_PROBE (opcode 8) returns a flat array of `io_uring_probe_op` — one bit per opcode: supported or not. That's it.

REGISTER_QUERY can answer richer questions:
- "Does IORING_OP_RECV support IORING_RECV_MULTISHOT on this kernel?"
- "Is IORING_REGISTER_BPF_FILTER available?"
- "What IOSQE flags are valid for this opcode?"

## Status

Fresh in 6.19. liburing support is being added. For now, most applications still use the probe API + feature flags + kernel version checks. REGISTER_QUERY will eventually replace all of that with a single query point.

## Recommendation

If you're targeting 6.19+, use REGISTER_QUERY. If you need to support older kernels (you do), keep the probe + feature flag fallback path. The two approaches aren't mutually exclusive.
