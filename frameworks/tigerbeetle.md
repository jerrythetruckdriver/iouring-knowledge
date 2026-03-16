# TigerBeetle

Financial transactions database written in Zig. The most serious io_uring deployment in the wild.

## Why It Matters

TigerBeetle chose io_uring as its sole I/O mechanism. Not as an optional backend — as the foundation. Everything goes through the ring: disk I/O, networking, timers.

## Architecture

- **Language**: Zig (zero-overhead abstractions, no hidden allocations)
- **I/O**: io_uring exclusively on Linux, IOCP on Windows, kqueue on macOS
- **Design**: Single-threaded event loop with io_uring
- **Storage**: Direct I/O with registered buffers

## io_uring Usage

- Registered buffers for all disk I/O (pre-pinned, zero-copy to/from NVMe)
- Registered files for all open file descriptors
- Linked operations for write-ahead log entries
- `IOPOLL` mode for storage completions on supported hardware
- Custom completion handling — no liburing, direct ring manipulation

## Design Decisions

TigerBeetle's Zig io_uring implementation (`src/io/io_uring.zig`) is notable for:

1. **No dynamic allocation in the I/O path** — all buffers pre-allocated
2. **Deterministic behavior** — fixed-size SQ/CQ, bounded resource usage
3. **Crash consistency** — io_uring's ordering guarantees used for WAL integrity
4. **Cross-platform abstraction** — same interface wraps io_uring/IOCP/kqueue

## Performance Claims

TigerBeetle targets 1M transactions/sec on commodity hardware. The io_uring integration is critical — traditional syscall-per-I/O approaches can't sustain this rate.

## Source

- Repository: https://github.com/tigerbeetle/tigerbeetle
- io_uring implementation: `src/io/io_uring.zig`
