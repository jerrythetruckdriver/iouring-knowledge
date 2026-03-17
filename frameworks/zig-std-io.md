# Zig std.Io — io_uring as a First-Class Backend

Zig is the first major language to make io_uring a swappable I/O backend in its standard library. Not a third-party crate. Not an optional plugin. The standard library.

## Timeline

- **Zig 0.15.x**: `std.Io` interface designed with pluggable backends in mind
- **Feb 13, 2026**: `std.Io.Evented` lands with io_uring and Grand Central Dispatch implementations
- **Target**: Zig 0.16.0 release

## Architecture

`std.Io` is an interface. Your application code is backend-agnostic:

```zig
fn app(io: std.Io) !void {
    try std.Io.File.stdout().writeStreamingAll(io, "Hello, World!\n");
}
```

Swap the backend at the callsite:

| Backend | Platform | Model |
|---------|----------|-------|
| `std.Io.Threaded` | All | Thread pool, classic syscalls |
| `std.Io.Evented` (io_uring) | Linux | Userspace stack switching (fibers) |
| `std.Io.Evented` (GCD) | macOS | Grand Central Dispatch + fibers |

The io_uring backend uses `IORING_SETUP_COOP_TASKRUN | IORING_SETUP_SINGLE_ISSUER`, 64-entry ring. Single `io_uring_enter` for submit+wait.

## How It Works

The evented implementation uses **stackful coroutines** (fibers/green threads). When a fiber issues I/O, it submits an SQE and yields. The event loop calls `io_uring_enter` to submit and wait, then resumes fibers whose CQEs arrived.

Strace of a hello world:

```
io_uring_setup(64, {flags=IORING_SETUP_COOP_TASKRUN|IORING_SETUP_SINGLE_ISSUER, ...}) = 3
io_uring_enter(3, 1, 1, IORING_ENTER_GETEVENTS, NULL, 8) = 1
```

Two syscalls for the entire I/O lifecycle. The threaded equivalent needs `writev`.

## Current Status (March 2026)

Experimental. Known issues:

- Performance degradation when building the Zig compiler itself with `IoMode.evented`
- Some functions still unimplemented
- Error handling needs work
- Needs `@maxStackSize` builtin for practical fiber sizing when overcommit is off

But the Zig compiler *does compile and work* using the io_uring backend.

## Why This Matters

Every other language bolts io_uring on as an afterthought:
- **Rust**: tokio-uring, monoio, compio — all third-party, all with different APIs
- **Go**: Fundamental runtime impedance mismatch with goroutines
- **Java**: Netty io_uring transport — framework-level, not stdlib
- **Node.js**: Nothing
- **Python**: Bindings only

Zig made it a first-class citizen. Write your app once, swap the I/O engine at init time. That's how it should be done.

## Design Decisions

1. **Fibers over async/await**: Zig removed async/await in 0.12.0. Fibers provide equivalent functionality without function coloring.
2. **SINGLE_ISSUER**: One ring per event loop thread. No cross-thread submission contention.
3. **COOP_TASKRUN**: Avoids IPI reschedules. Work processed when task transitions to kernel anyway.
4. **No SQPOLL**: Appropriate for general-purpose stdlib use. SQPOLL is a specialized optimization.

## Source

- [io_uring implementation PR](https://codeberg.org/ziglang/zig/pulls/31158)
- [GCD implementation PR](https://codeberg.org/ziglang/zig/pulls/31198)
- [Andrew Kelley's devlog announcement](https://ziglang.org/devlog/2026/)
