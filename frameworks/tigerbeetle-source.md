# TigerBeetle io_uring Source Analysis

Deep dive into `src/io/linux.zig` — how TigerBeetle actually uses io_uring.

## Architecture

TigerBeetle wraps Zig's `std.os.linux.IoUring` in a completion-based I/O layer (`IO` struct). No liburing dependency. No abstractions they don't control.

Key design: **three queues, not two.**

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  unqueued    │────▶│  SQ (kernel)│────▶│  completed   │
│  (overflow)  │     │             │     │  (callbacks) │
└─────────────┘     └─────────────┘     └─────────────┘
```

- `unqueued` — operations waiting for SQ space. Overflow buffer.
- Kernel SQ/CQ — standard io_uring rings.
- `completed` — CQEs reaped but callbacks not yet fired.

This separation prevents recursion. Callbacks never run inside `flush_completions()`.

## Completion Struct

Every I/O operation is a `Completion`:

```zig
pub const Completion = struct {
    io: *IO,
    result: i32 = undefined,
    operation: Operation,          // tagged union of all op types
    context: ?*anyopaque,          // type-erased callback context
    callback: *const fn(...) void, // type-erased callback
    awaiting_back: ?*Completion,   // doubly-linked list for cancel_all
    awaiting_next: ?*Completion,
};
```

The `awaiting` doubly-linked list tracks every in-flight operation. Exists solely for `cancel_all()` — graceful shutdown needs to cancel every outstanding op.

## Supported Operations

TigerBeetle uses a minimal subset — only what a database needs:

| Operation | Purpose |
|-----------|---------|
| `read` | WAL/data file reads with offset |
| `write` | WAL/data file writes with offset |
| `fsync` | `IORING_FSYNC_DATASYNC` — data integrity |
| `openat` | File open with `O_CLOEXEC` forced |
| `close` | File descriptor cleanup |
| `accept` | Client connections |
| `connect` | Replica connections |
| `recv` | Network reads |
| `send` | Network writes with `MSG_NOSIGNAL` |
| `timeout` | Event loop timing |
| `statx` | File metadata |
| `cancel` | Operation cancellation |

No multishot. No registered buffers. No provided buffer rings. No zero-copy.

## Event Loop: `run_for_ns()`

The core loop uses **absolute CLOCK_MONOTONIC timeouts**:

```zig
pub fn run_for_ns(self: *IO, nanoseconds: u63) !void {
    const current_ts = posix.clock_gettime(posix.CLOCK.MONOTONIC);
    const timeout_ts = .{ .sec = current_ts.sec, .nsec = current_ts.nsec + nanoseconds };

    while (!etime) {
        // Submit timeout that cancels after 1 completion
        timeout_sqe.prep_timeout(&timeout_ts, 1, IORING_TIMEOUT_ABS);
        try self.flush(1, &timeouts, &etime);
    }
    // Busy-loop reap remaining timeouts (stack-allocated timespec)
    while (timeouts > 0) _ = try self.flush_completions(0, &timeouts, &etime);
}
```

Critical detail: `timeout_ts` lives on the stack. The busy-loop at the end ensures all timeouts referencing that stack frame complete before the function returns.

## Submission Flow

`flush()` is the heartbeat:

1. `flush_submissions()` — `submit_and_wait()`, handles `EAGAIN`/`CompletionQueueOvercommitted` by draining CQ first
2. `flush_completions()` — batch reap into `[256]io_uring_cqe` stack buffer
3. Drain `unqueued` into SQ (copy-then-reset to avoid infinite loop)
4. Run `completed` callbacks

The unqueued→SQ drain runs *before* callbacks to prevent starvation. New I/O from callbacks can't steal slots from overflow operations.

## Error Handling

Every operation maps kernel errnos to Zig error sets exhaustively. `EINTR` always re-enqueues:

```zig
.INTR => {
    completion.io.enqueue(completion);
    return;
},
```

`EAGAIN` on reads also re-enqueues — XFS can return `EAGAIN` even on blocking files without `RWF_NOWAIT`. They document this edge case in comments.

## Cancellation: `cancel_all()`

Graceful shutdown cancels operations one-by-one:

```
inactive → next → queued → wait → next → ... → done
```

State machine walks the `awaiting` list tail-to-head. For each operation:
1. Submit `IORING_OP_ASYNC_CANCEL` targeting it
2. Wait for cancellation CQE
3. If `ENOENT` (not running) — skip. If `EALREADY` (not interruptable) — wait for natural completion.

**TODO in their source**: They note that kernel ≥5.19 has `IORING_ASYNC_CANCEL_ALL` which would eliminate the linked list and one-by-one approach. Kernel ≥6.0 has `io_uring_register_sync_cancel` which would remove the queued stage. Neither adopted yet.

## Init Guard

```zig
if (version.order(.{ .major = 5, .minor = 5, .patch = 0 }) == .lt) {
    @panic("Linux kernel 5.5 or greater is required for io_uring OP_ACCEPT");
}
```

Minimum kernel 5.5. Error messages distinguish `SystemOutdated` (seccomp blocking) from `PermissionDenied` (sysctl `kernel.io_uring_disabled`).

## Design Decisions

**Why no multishot accept?** TigerBeetle accepts a known, small number of replica connections. Multishot's value is high-frequency accept scenarios.

**Why no registered buffers?** Their I/O pattern is large sequential WAL writes with pre-allocated buffers. The overhead of `io_uring_register_buffers()` setup vs. the per-op savings didn't justify complexity.

**Why no SQPOLL?** They call `run_for_ns()` with precise timing. SQPOLL's dedicated kernel thread would burn CPU between ticks without benefit.

**Why Zig's IoUring, not liburing?** Zero external dependencies. Zig's stdlib wrapper is thin enough. They control the entire stack.

## What They Got Right

1. **Overflow handling** — `unqueued` queue with fairness guarantees
2. **No callback recursion** — deferred execution via `completed` list
3. **Stack-safe CQE reaping** — fixed `[256]` buffer, loops until drained
4. **Defensive re-enqueue** — `EINTR`/`EAGAIN` handled at every operation
5. **Clean shutdown** — cancel_all state machine ensures no dangling ops

## What They Could Improve

1. Adopt `IORING_ASYNC_CANCEL_ALL` (5.19+) to simplify shutdown
2. Use `io_uring_register_sync_cancel` (6.0+) for synchronous cancel
3. Consider registered files for the WAL fd (hot path)
4. `buffer_limit()` caps at 0x7ffff000 — inherited from `pwrite()` limits, but io_uring doesn't have this constraint for `IORING_OP_READ`/`WRITE`
