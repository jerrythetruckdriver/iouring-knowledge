# io_uring and WebAssembly (WASI)

## Current State

No major WASI runtime uses io_uring directly. The WASI I/O model is still evolving.

### WASI Preview 1 (wasi_snapshot_preview1)

POSIX-like fd operations: `fd_read`, `fd_write`, `fd_seek`, `poll_oneoff`. All synchronous. `poll_oneoff` is the only async-ish primitive — it's basically poll(2).

No completion-based I/O. No batching. Each WASI call maps to one or more host syscalls.

### WASI Preview 2 (Component Model)

Introduces `wasi:io/streams` — a stream-based I/O model with `pollable` resources. Closer to async, but still readiness-based (poll semantics), not completion-based.

```
wasi:io/streams.input-stream
  read(len) → list<u8>
  subscribe() → pollable    // readiness notification

wasi:io/poll
  poll(list<pollable>) → list<u32>   // like poll(2)
```

No submission queue, no completion queue, no batching. The mental model is still "wait for ready, then do I/O."

## Runtime Implementations

| Runtime | I/O Backend | io_uring |
|---------|------------|----------|
| Wasmtime | tokio (epoll/kqueue) | ❌ |
| Wasmer | Custom (epoll/kqueue) | ❌ |
| WasmEdge | epoll | ❌ |
| wazero | Go runtime | ❌ |
| Spin (Fermyon) | Wasmtime + tokio | ❌ |

## Why No io_uring

1. **WASI I/O model is readiness-based**: The `pollable` abstraction mirrors poll/epoll, not io_uring's completion model
2. **Cross-platform requirement**: WASI targets Linux, macOS, Windows. io_uring is Linux-only
3. **Security model**: WASI capabilities restrict what guests can do. io_uring's shared memory rings don't fit WASI's capability-based security
4. **Abstraction gap**: WASI guests don't control I/O strategy — the runtime does

## Where io_uring Could Help

Not inside the Wasm guest, but in the **host runtime**:

### Storage Layer

Runtime could use io_uring for filesystem operations behind WASI fd_read/fd_write:

```
Guest: fd_read(fd, buf, len)
  → Host: io_uring_prep_read(sqe, host_fd, ...)
  → Batch multiple guest fd_reads into one io_uring submit
```

This is transparent to the guest.

### Network Layer

WASI sockets could be backed by io_uring in the runtime:

```
Guest: wasi:sockets/tcp.receive()
  → Host: multishot recv on host socket
  → Buffer completions, deliver to guest on next poll
```

### Component Model Async

The Component Model's `async` canonical ABI (proposed) would map naturally to io_uring:

```
Guest: [async] wasi:filesystem/read(fd, len) → future<list<u8>>
  → Host: io_uring submit
  → Guest continues execution
  → Host: CQE arrives, resolves future
```

This is where WASI and io_uring could converge — if the async ABI proposal advances.

## The Gap

WASI's I/O primitives are too high-level for io_uring to surface directly. You can't give a Wasm guest access to SQ/CQ shared memory — it breaks the sandbox model.

The right approach:
1. Runtime uses io_uring internally (transparent to guest)
2. WASI async ABI maps to completion model under the hood
3. Guest code is platform-agnostic, runtime picks optimal backend

## Prediction

- Short term: No WASI runtime will expose io_uring to guests
- Medium term: Runtimes may use io_uring internally for host I/O
- Long term: WASI async ABI + io_uring backend is the natural convergence, but the async ABI is still in proposal stage
