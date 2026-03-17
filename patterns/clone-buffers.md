# Clone Buffers (IORING_REGISTER_CLONE_BUFFERS)

Opcode 30. Share registered buffer tables between rings without re-registering.

## The Problem

Registered buffers (`IORING_REGISTER_BUFFERS`) pin pages and create kernel-side buffer maps. This is expensive — page pinning, DMA mapping, kernel memory allocation. If you have multiple rings (e.g., one per thread), re-registering the same buffers for each ring wastes memory and time.

## The Solution

```c
struct io_uring_clone_buffers {
    __u32 src_fd;    // source ring fd
    __u32 flags;     // IORING_REGISTER_SRC_REGISTERED, IORING_REGISTER_DST_REPLACE
    __u32 src_off;   // offset in source buffer table
    __u32 dst_off;   // offset in destination buffer table
    __u32 nr;        // number of buffers to clone
    __u32 pad[3];
};
```

Register via:
```c
io_uring_register(dst_ring_fd, IORING_REGISTER_CLONE_BUFFERS,
                  &clone_args, sizeof(clone_args));
```

## Flags

| Flag | Value | Description |
|------|-------|-------------|
| `IORING_REGISTER_SRC_REGISTERED` | 1 << 0 | `src_fd` is a registered ring fd, not a regular fd |
| `IORING_REGISTER_DST_REPLACE` | 1 << 1 | Replace existing buffers at `dst_off` in destination |

## Partial Clone

You don't have to clone the entire table. Use `src_off`, `dst_off`, and `nr` to clone a subset:

```c
// Clone buffers 10-19 from source ring, place at offset 0 in dest
clone_args.src_off = 10;
clone_args.dst_off = 0;
clone_args.nr = 10;
```

## Use Cases

### Thread-Per-Core with Shared Buffers

```
Thread 0: ring_0 → registers buffers (expensive, one-time)
Thread 1: ring_1 → clones from ring_0 (cheap, shares mappings)
Thread 2: ring_2 → clones from ring_0
...
```

All threads share the same pinned page mappings. Huge memory savings for large buffer pools.

### Ring Migration

Moving work from one ring to another (e.g., rebalancing, ring resize) without re-registering:

```c
// New ring gets a copy of the old ring's buffers
clone_args.src_fd = old_ring_fd;
clone_args.flags = 0;
clone_args.src_off = 0;
clone_args.dst_off = 0;
clone_args.nr = 0;  // 0 = clone all
```

## Kernel Version

- **6.10**: `IORING_REGISTER_CLONE_BUFFERS` added

## Gotchas

1. Source ring must have registered buffers. Destination range must not overlap with existing registrations (unless `IORING_REGISTER_DST_REPLACE` is set).
2. Cloned buffers share the underlying page pins. Unregistering from one ring doesn't affect the other.
3. Both rings must be in the same process (same `task_struct`).
