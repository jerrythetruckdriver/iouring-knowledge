# io_uring ABI Stability

## The Contract

io_uring's userspace ABI is covered by the kernel's standard ABI stability guarantee: **once a feature is in a stable release, it won't break**.

This means:
- Struct layouts are frozen once shipped
- Opcode numbers are permanent
- Flag values don't change
- `io_uring_setup()`, `io_uring_enter()`, `io_uring_register()` syscall numbers are stable

## What's Stable

### Syscall Interface

Three syscalls, numbers fixed per architecture:

| Syscall | x86_64 | Notes |
|---------|--------|-------|
| `io_uring_setup` | 425 | Returns ring fd |
| `io_uring_enter` | 426 | Submit + wait |
| `io_uring_register` | 427 | Register resources |

### Struct Layouts

All structs in `include/uapi/linux/io_uring.h` are ABI:

- `io_uring_sqe` — 64 bytes (or 128 with SQE128), layout frozen
- `io_uring_cqe` — 16 bytes (or 32 with CQE32), layout frozen
- `io_uring_params` — returned by setup, layout frozen
- `io_sqring_offsets` / `io_cqring_offsets` — mmap offset structs, frozen

New fields use reserved (`resv`) padding. Example: `io_uring_params` has `__resv[3]` — future features can repurpose these without changing struct size.

### Opcodes

`enum io_uring_op` values are permanent:
- `IORING_OP_NOP` = 0 (forever)
- `IORING_OP_READV` = 1 (forever)
- New opcodes only append to the enum
- `IORING_OP_LAST` is the sentinel, not a valid opcode

### Register Opcodes

`enum io_uring_register_op` values are permanent:
- `IORING_REGISTER_BUFFERS` = 0 (forever)
- New register ops only append
- `IORING_REGISTER_LAST` is the sentinel

### Flags

Setup flags, SQE flags, CQE flags, timeout flags — all bit positions are permanent once released.

## What Can Change

### New Additions (Backwards Compatible)

- New opcodes (appended to enum)
- New register operations (appended to enum)
- New flags in existing flag spaces (using previously-reserved bits)
- New fields in structs (using reserved padding)
- New feature flags in `io_uring_params.features`

### Behavioral Refinements

- Error codes for edge cases may be clarified
- Performance characteristics change between kernel versions
- io-wq thread pool behavior has been refined multiple times
- CQ overflow handling was improved (IORING_FEAT_NODROP)

### What Won't Change

- Existing opcode semantics for documented behavior
- SQE/CQE struct sizes and field offsets
- Mmap offset constants
- Syscall numbers

## Feature Detection Strategy

The right way to handle kernel version differences:

### 1. Probe API (Pre-6.19)

```c
struct io_uring_probe *probe = io_uring_get_probe();
if (io_uring_opcode_supported(probe, IORING_OP_RECV_ZC))
    // Use zero-copy receive
```

### 2. REGISTER_QUERY (6.19+)

```c
struct io_uring_query_op ops[IORING_OP_LAST];
io_uring_register(ring_fd, IORING_REGISTER_QUERY, ops, IORING_OP_LAST);
```

More powerful — queries opcode support, zcrx features, ring layout.

### 3. Feature Flags

```c
struct io_uring_params p = {};
int fd = io_uring_setup(entries, &p);
if (p.features & IORING_FEAT_RECVSEND_BUNDLE)
    // Bundle ops available
```

### 4. Trial and Error

For setup flags, submit with the flag and check for `EINVAL`. This is actually the recommended approach for setup flags that don't have corresponding feature flags.

## liburing ABI

liburing provides a stable C API that abstracts minor kernel differences:

- **SO version**: liburing.so.2 (major version 2 since liburing-2.0)
- **API additions**: new `io_uring_prep_*` helpers appear with each release
- **Backwards compatible**: code compiled against liburing 2.0 works with 2.14
- **Header-only inlines**: most prep helpers are inline functions in the header

liburing versions don't correspond 1:1 to kernel versions. liburing 2.14 includes helpers for kernel 6.19 features, but those helpers just prepare SQEs — they work on any kernel (the kernel will return EINVAL for unsupported ops).

## Application Strategy

```
1. io_uring_setup() — if ENOSYS, fall back entirely
2. Check params.features for needed feature flags
3. Probe for needed opcodes
4. Set up tiered backend:
   - Tier 1: Full io_uring (all features available)
   - Tier 2: Basic io_uring (core ops only)
   - Tier 3: epoll fallback
```

## Comparison with Other Kernel ABIs

| Interface | ABI Stability | Feature Detection |
|-----------|--------------|-------------------|
| io_uring | Strong (uapi header) | Probe, features, query |
| epoll | Strong (stable since 2.6) | Compile-time only |
| io_submit (AIO) | Strong | None (fixed feature set) |
| BPF | Stable syscall, evolving verifier | BPF_PROG_TYPE check |
| ioctl | Per-driver, varies | Trial and error |

io_uring's ABI stability story is unusually good for a rapidly evolving subsystem. The reserved-field-with-feature-detection pattern lets it grow without breaking existing code.
