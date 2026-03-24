# Go + io_uring via cgo and Pure Go

## The Go Problem

Go's goroutine scheduler multiplexes goroutines onto OS threads via netpoller (epoll on Linux). The runtime owns the event loop. You can't replace it with io_uring without forking the runtime.

This means Go programs can't benefit from io_uring for networking without fighting the runtime. File I/O is a different story — Go already offloads blocking file ops to thread pools.

## Approaches

### 1. cgo Wrappers (liburing bindings)

Call liburing functions from Go via cgo.

**Overhead per cgo call**: ~100-200ns. For batched submissions (one `io_uring_submit()` per N ops), this is amortized. For per-op cgo crossings, it destroys the benefit.

**Libraries**:
- None actively maintained at scale

**Problems**:
- cgo calls are not goroutine-friendly (pin OS thread during call)
- Buffer management across Go GC / C heap boundary
- Go's garbage collector can't track C-allocated memory
- Debugging is painful (mixed stacks)

### 2. Pure Go via syscall (no cgo)

Direct `mmap` + `io_uring_setup` + `io_uring_enter` syscalls from Go. No cgo overhead.

**Libraries**:
- **giouring** (pawelgaczynski/giouring): Pure Go port of liburing API. Full struct/function coverage. Powers the [Gain](https://github.com/pawelgaczynski/gain) networking framework. Most complete implementation.
- **iouring-go** (iceber/iouring-go): Higher-level API with channels for completion delivery. File + socket I/O, timeouts, linked requests, SQPOLL. Go-idiomatic but less complete.

**The giouring approach**:
```go
ring, _ := giouring.CreateRing(256)
defer ring.QueueExit()

sqe := ring.GetSQE()
sqe.PrepRecv(fd, buf, len, 0)
sqe.SetData64(requestID)

ring.Submit()

var cqe *giouring.CompletionQueueEvent
ring.WaitCQE(&cqe)
result := cqe.Res()
ring.CQESeen(cqe)
```

### 3. Gain Framework (Pure Go + io_uring networking)

Built on giouring. Thread-per-core architecture with one ring per worker goroutine. Pinned goroutines via `runtime.LockOSThread()`.

Event loop runs outside Go's netpoller entirely. Goroutines that interact with Gain don't use standard `net.Conn` — it's a separate networking stack.

## Impedance Mismatch

| Go Assumption | io_uring Reality |
|--------------|-----------------|
| GC manages all memory | Registered buffers must be pinned |
| Goroutines are cheap, block freely | Ring completion is callback-oriented |
| net.Conn is the abstraction | Direct fd operations |
| Runtime owns epoll | You need your own event loop |
| Preemptive scheduling | SINGLE_ISSUER wants thread affinity |

## Buffer Ownership

Go's GC can move objects. io_uring needs stable addresses for:
- Registered buffers (pinned with `IORING_REGISTER_BUFFERS`)
- Provided buffer rings (mmap'd)
- SQE addr fields pointing to data

Solutions:
- `mmap` for all I/O buffers (outside GC)
- `cgo.NewHandle` for Go→C references
- Never pass GC-managed `[]byte` directly to SQE addr fields

## When Go + io_uring Makes Sense

| Use Case | Verdict |
|----------|---------|
| HTTP server (standard library) | No — Go's netpoller is fine |
| Custom protocol, high connection count | Maybe — Gain shows 2-3x improvement |
| File I/O heavy (database, storage) | Yes — bypass Go's thread pool |
| Proxy / gateway | Maybe — if you're willing to leave net.Conn |
| CLI tool | No — complexity not justified |

## The Honest Take

Go and io_uring are architecturally mismatched. The runtime assumes it owns I/O scheduling. Working around this requires `LockOSThread()`, custom buffer management, and abandoning the standard library's networking abstractions.

If you need io_uring performance, you probably shouldn't be writing in Go. Use Rust (monoio, glommio), Zig (std.Io), or C. If you must use Go, giouring/Gain is the least painful path.

libuv (which powers Node.js) uses io_uring only for file I/O — 13 opcodes, no networking. That's the pragmatic ceiling for runtime-managed languages.
