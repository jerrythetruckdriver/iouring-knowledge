# IORING_REGISTER_CLOCK

Register opcode 29. Override the clock source used by io_uring timeouts.

## Default Behavior

io_uring uses `CLOCK_MONOTONIC` for all timeout operations by default. Individual timeouts can request `CLOCK_BOOTTIME` or `CLOCK_REALTIME` via `IORING_TIMEOUT_BOOTTIME` / `IORING_TIMEOUT_REALTIME` flags.

## The Problem

Setting clock flags per-timeout is tedious when your entire application uses one clock. Worse, if you want `CLOCK_MONOTONIC_RAW` or a custom clock, there was no way to specify it.

## Registration

```c
struct io_uring_clock_register {
    __u32 clockid;
    __u32 __resv[3];
};

struct io_uring_clock_register reg = {
    .clockid = CLOCK_BOOTTIME,
};
io_uring_register(ring_fd, IORING_REGISTER_CLOCK, &reg, 1);
```

After registration, all timeouts on this ring default to the specified clock. Per-timeout flags still override.

## When to Use

- **Containers/VMs:** `CLOCK_BOOTTIME` avoids time jumps from suspend/resume
- **Registered wait:** `io_uring_reg_wait` uses the registered clock for `min_wait_usec` and timeout calculations
- **Consistency:** Set once, forget — no per-timeout flag ceremony

## Supported Clocks

Any `clockid_t` the kernel supports:
- `CLOCK_MONOTONIC` (default)
- `CLOCK_BOOTTIME`
- `CLOCK_REALTIME`
- Others as supported by the running kernel

## Interaction with Registered Wait

`IORING_REGISTER_CLOCK` + `IORING_REGISTER_MEM_REGION` (for reg_wait) is the modern high-performance timeout path. Zero syscall overhead for wait parameters, consistent clock source.
