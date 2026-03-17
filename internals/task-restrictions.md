# Task Restrictions

Per-task opcode and flag restrictions for io_uring rings.

## Two Layers

### Ring-Level: IORING_REGISTER_RESTRICTIONS (opcode 11)

The original restrictions API. Applied to the ring at setup time (before `IORING_REGISTER_ENABLE_RINGS`). Once enabled, immutable.

```c
struct io_uring_restriction res[] = {
    { .opcode = IORING_RESTRICTION_SQE_OP,      .sqe_op = IORING_OP_READ },
    { .opcode = IORING_RESTRICTION_SQE_OP,      .sqe_op = IORING_OP_WRITE },
    { .opcode = IORING_RESTRICTION_REGISTER_OP,  .register_op = IORING_REGISTER_FILES },
    { .opcode = IORING_RESTRICTION_SQE_FLAGS_ALLOWED, .sqe_flags = IOSQE_FIXED_FILE },
};
io_uring_register(ring_fd, IORING_REGISTER_RESTRICTIONS, res, 4);
io_uring_register(ring_fd, IORING_REGISTER_ENABLE_RINGS, NULL, 0);
```

After enable: only READ, WRITE allowed. Only FIXED_FILE flag permitted. Only FILES registration.

Three restriction types:
- `IORING_RESTRICTION_SQE_OP` — allow specific opcode
- `IORING_RESTRICTION_SQE_FLAGS_ALLOWED` — allow specific SQE flags
- `IORING_RESTRICTION_SQE_FLAGS_REQUIRED` — require specific flags on every submission
- `IORING_RESTRICTION_REGISTER_OP` — allow specific register opcodes

### BPF Filtering: IORING_REGISTER_BPF_FILTER (opcode 37, 6.19)

Programmable per-SQE filtering. A BPF program inspects each SQE and returns allow/deny. More flexible than the static restrictions API — can make decisions based on fd values, buffer ranges, flag combinations.

See: [pitfalls/bpf-filtering.md](../pitfalls/bpf-filtering.md)

## Task-Level: io_uring_task_restriction

```c
struct io_uring_task_restriction {
    __u16 flags;
    __u16 nr_res;
    __u32 resv[3];
    struct io_uring_restriction restrictions[0];
};
```

Per-task restrictions extend the ring-level model. A privileged process sets up the ring with broad permissions, then attaches per-task restrictions before handing the ring fd to less-privileged threads.

## Use Cases

- **Sandboxing:** Give a plugin access to a ring but restrict it to READ + WRITE on fixed files only
- **Multi-tenant:** Different workers get different opcode allowlists on a shared ring
- **Defense in depth:** Restrict ring capabilities even if the process is compromised

## Limitations

- Ring restrictions are set-once, immutable after enable
- No per-fd restrictions (can't say "only read from fd 5")
- BPF filtering is more flexible but requires 6.19+
- Restrictions don't compose — if ring allows READ and task allows WRITE, the intersection is empty (nothing allowed)

## Production Guidance

For new deployments on 6.19+: use BPF filtering. It's strictly more powerful than static restrictions and can be updated.

For older kernels: ring-level restrictions + `IORING_SETUP_R_DISABLED` startup pattern. Set restrictions before enabling the ring.

For maximum security: combine with seccomp (restrict `io_uring_setup`/`io_uring_enter` syscalls) and sysctl `kernel.io_uring_disabled`.
