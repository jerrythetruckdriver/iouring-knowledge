# io_uring-go

Go bindings for io_uring. The Go runtime's goroutine model makes io_uring integration... interesting.

## The Go Problem

Go's runtime has its own I/O scheduler (netpoller) built on epoll. Goroutines park on I/O and the runtime handles scheduling. io_uring doesn't fit this model natively because:

1. Go's GC can move memory — registered buffers need to be pinned
2. Goroutine scheduling is invisible to io_uring
3. cgo calls have overhead that partially negates io_uring's syscall reduction

## Approaches

### pawelgaczynski/giouring
- Low-level bindings via cgo
- Thin wrapper around liburing
- User manages ring directly

### iceber/iouring-go
- Pure Go implementation (no cgo)
- Direct syscall interface
- Avoids cgo overhead
- More idiomatic Go API

### dshulyak/uring
- Pure Go, performance-focused
- Ring buffer management in Go
- Minimal allocations

## Performance Considerations

The cgo boundary costs ~100ns per call. If you're using io_uring to save ~1-5µs per syscall, cgo eats a significant chunk of that.

Pure Go implementations avoid this but must handle:
- Buffer pinning (prevent GC from moving io_uring buffers)
- Memory barriers (Go's memory model vs io_uring's)
- Goroutine integration (waking parked goroutines on completion)

## Verdict

io_uring in Go is a hard sell. Go's netpoller is already quite good, and the impedance mismatch with Go's runtime is real. Best use case: storage I/O where Go's netpoller doesn't help.

## Source

- giouring: https://github.com/pawelgaczynski/giouring
- iouring-go: https://github.com/iceber/iouring-go
