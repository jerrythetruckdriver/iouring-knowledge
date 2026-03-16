# io_uring BPF Filtering

`IORING_REGISTER_BPF_FILTER` (register opcode 37). Added in the 6.19 development cycle.

## Purpose

Attach BPF programs to an io_uring instance to filter which operations are allowed. This is the proper sandboxing mechanism for io_uring — the old restrictions API (`IORING_REGISTER_RESTRICTIONS`) was limited to static allow/deny lists.

## Why It's Needed

io_uring bypasses seccomp. That's by design — the whole point is to avoid syscall overhead. But it means:

1. Container runtimes can't use seccomp to restrict io_uring operations
2. `IORING_REGISTER_RESTRICTIONS` only allows/denies entire opcodes — no argument inspection
3. No way to say "allow reads but only to these paths" or "allow sends but only to this IP range"

BPF filtering fixes all of this. The BPF program can inspect the SQE contents and make fine-grained decisions.

## How It Works

1. Write a BPF program that takes an `io_uring_sqe` as input
2. Program returns allow/deny per submission
3. Register the program with `IORING_REGISTER_BPF_FILTER`
4. Every SQE submission passes through the filter

The BPF program can inspect:
- Opcode
- File descriptor
- Flags
- Buffer addresses (though you can't dereference userspace pointers in BPF)
- Offset, length, etc.

## vs. Restrictions API

| | Restrictions API | BPF Filtering |
|---|---|---|
| Granularity | Per-opcode | Per-SQE field |
| Dynamic | No — set once at ring creation | Yes — BPF program can encode complex logic |
| Kernel version | 5.12+ | 6.19+ |
| Overhead | Near-zero (bitmap check) | BPF execution per SQE (nanoseconds) |

## vs. Seccomp

| | Seccomp | BPF Filter |
|---|---|---|
| Scope | Syscall boundary | io_uring SQE boundary |
| io_uring coverage | Only `io_uring_enter`/`io_uring_setup` | Every submitted operation |
| Overhead | Per-syscall | Per-SQE |

## Security Model

BPF filtering is the answer to "io_uring is a security nightmare." The complaints were valid:
- CVEs in io_uring code (it's complex, bugs happen)
- seccomp bypass by design
- Container runtimes disabling io_uring entirely (Docker, Android)

With BPF filtering:
- Container runtimes can allow io_uring with fine-grained restrictions
- No need to disable io_uring wholesale
- Security policy is as expressive as BPF allows

## Current State

The header defines `IORING_REGISTER_BPF_FILTER = 37`. Implementation is in the 6.19 development tree. Expect the API to stabilize quickly — it's a blocker for container adoption.

## Recommendation

If you're running io_uring in a multi-tenant environment: wait for BPF filtering to land and mature. If you're running single-tenant (your code, your server): the restrictions API is sufficient, or just don't bother — you trust your own code.
