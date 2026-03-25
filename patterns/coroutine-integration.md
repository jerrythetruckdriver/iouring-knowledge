# Completion-Based Coroutine/Fiber Integration

## The Problem

Readiness-based I/O (epoll): "the fd is ready, now do the syscall." Works naturally with stackful/stackless coroutines that yield at readiness checks.

Completion-based I/O (io_uring): "here's the result of the I/O you submitted." Requires yielding at submission and resuming at completion. This changes coroutine design fundamentally.

## C++20 Coroutines

The cleanest integration. `co_await` an io_uring operation:

```cpp
task<ssize_t> async_read(io_uring *ring, int fd, void *buf, size_t len) {
    struct completion {
        std::coroutine_handle<> handle;
        int result;
    } comp;

    auto *sqe = io_uring_get_sqe(ring);
    io_uring_prep_read(sqe, fd, buf, len, 0);
    io_uring_sqe_set_data(sqe, &comp);
    io_uring_submit(ring);

    // Suspend coroutine. Will be resumed when CQE arrives.
    co_await suspend_always{};

    co_return comp.result;
}

// Event loop processes CQEs and resumes coroutines:
void event_loop(io_uring *ring) {
    struct io_uring_cqe *cqe;
    while (true) {
        io_uring_wait_cqe(ring, &cqe);
        auto *comp = (completion *)io_uring_cqe_get_data(cqe);
        comp->result = cqe->res;
        comp->handle.resume();  // resume the coroutine
        io_uring_cqe_seen(ring, cqe);
    }
}
```

Key insight: `user_data` points to the coroutine handle. CQE arrives → resume coroutine → coroutine reads result → continues execution.

## Zig Fibers (std.Io.Evented)

Zig's approach (landed Feb 2026):

```zig
// Zig fibers suspend on io_uring submission, resume on completion
// The runtime manages the fiber↔CQE mapping internally

const io = try std.Io.Evented.init(.{});
defer io.deinit();

// This suspends the fiber, submits to io_uring, resumes on completion
const bytes_read = try io.read(fd, buffer);
```

Under the hood: Zig creates a ring with `COOP_TASKRUN | SINGLE_ISSUER`, 64 entries. Fibers (green threads) yield to the event loop which calls `io_uring_submit_and_wait`. CQEs resume the corresponding fiber.

## Rust: The Ownership Problem

Rust's `AsyncRead`/`AsyncWrite` traits take `&mut [u8]` — borrowed buffers. io_uring needs ownership (the kernel accesses the buffer after the function returns).

```rust
// Tokio/async-std pattern (readiness-based):
async fn read(&mut self, buf: &mut [u8]) -> io::Result<usize>;
// Fine for epoll: buf is only used during the synchronous read() syscall

// io_uring needs:
async fn read(self, buf: Vec<u8>) -> (io::Result<usize>, Vec<u8>);
// Must take ownership of buf and return it — buffer is in-flight until CQE
```

This is why `tokio-uring`, `monoio`, and `compio` have their own traits:
- **monoio**: `AsyncReadRent` / `AsyncWriteRent` — takes ownership, returns buffer
- **compio**: `AsyncBufRead` — similar ownership transfer
- **glommio**: custom I/O traits with buffer pools

No drop-in replacement for Tokio async traits. You're choosing a different ecosystem.

## Go: goroutine + io_uring

Go's goroutine model parks goroutines on I/O. With io_uring, you'd submit from one goroutine and resume others on completion:

```go
// Conceptual — actual implementations vary
func Read(ring *Ring, fd int, buf []byte) (int, error) {
    ch := make(chan result, 1)
    sqe := ring.GetSQE()
    sqe.PrepRead(fd, buf)
    sqe.SetUserData(uintptr(unsafe.Pointer(&ch)))
    ring.Submit()

    // Park goroutine until CQE arrives
    res := <-ch
    return res.n, res.err
}
```

Problem: Go's GC can move `buf` while the kernel is reading into it. Solutions:
- Pin with `runtime.KeepAlive` (doesn't prevent GC movement)
- Allocate outside Go heap (cgo malloc)
- Use registered buffers (kernel pins the pages)

This is why Go + io_uring is awkward. The GC doesn't know about in-flight kernel references.

## Event Loop Pattern

All completion-based coroutine systems share this structure:

```
1. Coroutine calls async I/O function
2. Function prepares SQE, stores coroutine handle in user_data
3. Function yields/suspends
4. Event loop: io_uring_submit_and_wait()
5. For each CQE:
   a. Extract coroutine handle from user_data
   b. Store result
   c. Resume coroutine
6. Coroutine reads result, continues execution
7. Goto 1
```

Batch submission is natural — multiple coroutines prepare SQEs before the event loop submits them all.

## Multishot Complications

Multishot operations (accept, recv) produce multiple CQEs per SQE. The coroutine doesn't "complete" — it produces a stream:

```cpp
// Multishot accept as async generator
async_generator<int> accept_loop(io_uring *ring, int listen_fd) {
    auto *sqe = io_uring_get_sqe(ring);
    io_uring_prep_multishot_accept(sqe, listen_fd, NULL, NULL, 0);
    sqe->user_data = /* this coroutine */;

    while (true) {
        co_yield /* next CQE's fd */;  // yield each accepted fd
        // Coroutine stays alive, waits for next CQE
        // Until CQE without CQE_F_MORE terminates the multishot
    }
}
```

The coroutine resume/suspend cycle maps 1:1 with CQE delivery.

## Framework Comparison

| Framework | Language | Coroutine Type | io_uring Integration |
|-----------|----------|---------------|---------------------|
| Seastar | C++ | Futures/continuations | SQE submit → future resolve |
| Zig std.Io | Zig | Green threads (fibers) | Fiber suspend/resume |
| monoio | Rust | async/await (owned bufs) | CQE → task wake |
| glommio | Rust | async/await (custom traits) | CQE → task wake |
| compio | Rust | async/await (BufResult) | CQE → task wake |
| TigerBeetle | Zig | Callbacks (no coroutines) | CQE → callback dispatch |
| QEMU | C | Coroutines (custom) | CQE → coroutine resume |
| FoundationDB | C++ | Actor model (Flow) | Boost.Asio io_uring → future |

The cleanest integrations are purpose-built (monoio, glommio, Zig std.Io). Retrofitting io_uring into existing coroutine systems (Tokio, Go goroutines) always has friction.
