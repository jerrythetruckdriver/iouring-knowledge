# Wasm Runtimes and io_uring

## Status: None Use io_uring (as of March 2026)

| Runtime | Language | I/O Model | io_uring |
|---------|----------|-----------|----------|
| Wasmtime | Rust | tokio (epoll) | No |
| Wasmer | Rust | std (epoll) | No |
| WasmEdge | C++ | epoll | No |
| wasm3 | C | minimal I/O | No |
| WAMR | C | libuv/epoll | No |

Zero io_uring references in any major Wasm runtime codebase. Confirmed via GitHub code search.

## Why Not

**Security model prevents guest ring access:**
Wasm's sandbox is the point. Giving a Wasm module direct access to io_uring rings would bypass the sandbox. Ring memory is shared between kernel and userspace — exactly the kind of thing Wasm isolates guests from.

**Host-side could use it transparently:**
Nothing prevents a runtime from using io_uring internally to service WASI I/O calls. Guest calls `fd_read()` → runtime submits `IORING_OP_READ` → returns result. The guest never knows. But no runtime does this yet.

**WASI model mismatch:**
- WASI P1: POSIX-like, synchronous `fd_read/fd_write`. Could be backed by io_uring transparently.
- WASI P2 (Component Model): readiness-based `pollable` type. More epoll-shaped than io_uring-shaped.
- WASI async ABI (proposed): completion-based. This is where io_uring alignment could happen.

## The Path Forward

1. WASI async ABI lands with completion-based I/O model
2. Runtimes implement it with io_uring as the Linux backend
3. Wasm modules get io_uring-level batching without knowing it

Timeline: don't hold your breath. WASI P2 is still stabilizing.

## Comparison with Native

The overhead of Wasm sandboxing (bounds checks, memory copies at boundary) dwarfs any I/O syscall savings io_uring would provide. If you need io_uring performance, run native code.

## See Also

- [WASI/WebAssembly Patterns](../patterns/wasi-webassembly.md) — detailed WASI model analysis
- [Edge Compute](edge-compute.md) — V8 isolates and io_uring
- [Redpanda Wasm](redpanda-wasm.md) — Wasm transforms don't touch I/O
