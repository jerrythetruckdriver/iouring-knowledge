# libuv, Node.js, and Bun: io_uring Status

## libuv (Node.js I/O Layer)

libuv uses io_uring for **file operations only**. Not networking.

### What libuv Submits via io_uring

From `libuv/src/unix/linux.c` (v1.x branch, March 2026):

```c
enum {
  UV__IORING_OP_READV = 1,
  UV__IORING_OP_WRITEV = 2,
  UV__IORING_OP_FSYNC = 3,
  UV__IORING_OP_OPENAT = 18,
  UV__IORING_OP_CLOSE = 19,
  UV__IORING_OP_STATX = 21,
  UV__IORING_OP_EPOLL_CTL = 29,    // epoll management, not I/O
  UV__IORING_OP_RENAMEAT = 35,
  UV__IORING_OP_UNLINKAT = 36,
  UV__IORING_OP_MKDIRAT = 37,
  UV__IORING_OP_SYMLINKAT = 38,
  UV__IORING_OP_LINKAT = 39,
  UV__IORING_OP_FTRUNCATE = 55,
};
```

That's filesystem ops. No `RECV`, no `SEND`, no `ACCEPT`.

### Architecture

Two separate rings:
- **`ctl` ring** — 256-entry, used for batching `EPOLL_CTL` operations via io_uring. Always initialized.
- **`iou` ring** — 64-entry, optional. Handles file I/O. Lazily created on first file operation.

SQPOLL is **opt-in** via `UV_LOOP_ENABLE_IO_URING_SQPOLL` flag AND `UV_USE_IO_URING=1` env var. Default: no SQPOLL.

Uses `IORING_SETUP_NO_SQARRAY` on kernels ≥ 6.6.

### Why No Networking?

libuv's networking model is epoll-based with per-fd callbacks. Rearchitecting for io_uring completion-based networking would:
- Break the entire watcher abstraction
- Require buffer ownership changes throughout Node.js
- Add platform-specific codepaths for every socket operation
- Provide marginal gains for typical Node.js workloads (which aren't syscall-bound)

The file operations benefit because they avoid the threadpool. That's the win.

### Platform Guards

```c
// Disabled entirely:
// - Android (seccomp blocks it)
// - 32-bit ARM (all 32-bit kernels appear buggy)
// - ppc64 (random SIGSEGV in signal handler)
// - parisc kernels < 6.1.51
```

SQPOLL requires kernel ≥ 5.10.186 (100% CPU bug in older kernels).
Close operations require kernel ≥ 5.15.90 or ≥ 6.1.0 (ETXTBSY bug under Docker).

## Node.js

Node.js inherits libuv's io_uring support passively. No Node.js-specific io_uring code.

What users get:
- `fs.readFile()`, `fs.writeFile()`, `fs.stat()`, `fs.rename()`, etc. go through io_uring on Linux ≥ 5.13
- `net.createServer()`, HTTP, WebSocket → still epoll
- No configuration needed (except SQPOLL)
- No way to access io_uring directly from JavaScript

### Impact

For I/O-heavy file workloads, io_uring avoids threadpool overhead. For networking (which is what most Node.js apps do), zero impact.

## Bun

Bun is written in Zig and uses its own event loop, **not** libuv.

From `src/io/io.zig` (March 2026):

```zig
// Confusingly, this is the barely used epoll/kqueue event loop
// This is only used by Bun.write() and Bun.file(path).text() & friends.
// Most I/O happens on the main thread.
```

Bun's I/O layer uses **epoll on Linux, kqueue on macOS**. No io_uring.

The `src/io/` directory contains:
- `io.zig` — epoll/kqueue event loop
- `io_darwin.cpp` — macOS-specific
- `PipeReader.zig`, `PipeWriter.zig` — pipe abstractions
- No io_uring files

### Why Not?

Bun prioritizes cross-platform consistency (macOS is a first-class target). Building an io_uring backend for Linux while maintaining kqueue for macOS doubles the I/O surface. Bun's Zig-based event loop is already fast enough that the epoll overhead isn't the bottleneck.

If Zig stdlib's `std.Io.Evented` (which has an io_uring backend since Feb 2026) matures, Bun could potentially adopt it. But that's speculative.

## Summary

| Runtime | io_uring? | Scope | Since |
|---------|-----------|-------|-------|
| libuv/Node.js | Yes | File ops only (13 opcodes) | libuv 1.45+ |
| Bun | No | epoll/kqueue | — |
| Deno | No | tokio (epoll) | — |

The JavaScript runtime ecosystem treats io_uring as a file I/O optimization, not a networking revolution. The completion-based model clashes with their event loop architectures.

## Sources

- libuv v1.x `src/unix/linux.c` (March 2026)
- Bun `src/io/io.zig` (March 2026)
- Bun `src/io/` directory listing
