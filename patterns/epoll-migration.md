# epoll → io_uring Migration

## IORING_OP_EPOLL_WAIT (6.15+)

Sounds backwards. io_uring has its own event model — why would you read epoll events through it?

Because real codebases aren't greenfield. You've got 50k lines of epoll event loop code. Some events migrate cleanly to io_uring (networking, file I/O). Others are stuck on readiness-based APIs (custom fd types, third-party libraries that only give you an fd to poll).

`IORING_OP_EPOLL_WAIT` lets you do partial migration:

```
Before:  epoll_wait() → handle network + handle timers + handle signals
After:   io_uring handles network (completion-based)
         io_uring EPOLL_WAIT handles legacy fds (readiness via completion)
         Single io_uring_enter() drives everything
```

One submission syscall instead of epoll_wait + io_uring_enter.

## Migration Strategy

### Phase 1: Wrap epoll in io_uring

Keep your epoll fd. Submit `IORING_OP_EPOLL_WAIT` to get epoll events via CQE. You've eliminated one syscall per loop iteration. Zero code change to event handlers.

### Phase 2: Move hot paths

Identify your highest-throughput I/O. Move accept, recv, send to native io_uring ops. Leave edge cases on epoll.

### Phase 3: Multishot everything

Replace one-shot accepts and receives with multishot variants. Now the kernel keeps delivering events without resubmission.

### Phase 4: Kill epoll

When all fds are handled natively by io_uring, drop the epoll fd entirely.

## The Full Async Socket Lifecycle (6.14+)

As of 6.14, you can do the entire TCP server lifecycle without a single synchronous syscall:

```
IORING_OP_SOCKET    → create socket (async)
IORING_OP_BIND      → bind to address (6.14)
IORING_OP_LISTEN    → start listening (6.14)
IORING_OP_ACCEPT    → accept connections (multishot)
IORING_OP_RECV      → receive data (multishot, zero-copy)
IORING_OP_SEND      → send data (zero-copy)
IORING_OP_CLOSE     → close connection
```

Before 6.14, you still needed synchronous `bind()` and `listen()` calls. Now the entire path is submittable through the ring.

## epoll vs io_uring: When epoll Still Wins

- **Portability**: epoll works on any Linux 2.6+. io_uring needs 5.1+ and keeps evolving.
- **Simplicity**: epoll is 3 syscalls. io_uring setup is complex.
- **Low connection count**: If you have < 100 fds, epoll's overhead is negligible.
- **Debugging**: epoll behavior is well-understood. io_uring failure modes are still being catalogued.

## Syscall Comparison

| Operation | epoll | io_uring |
|---|---|---|
| Wait for events | `epoll_wait()` — 1 syscall | Batched in `io_uring_enter()` |
| Accept connection | `accept4()` — 1 syscall per connection | Multishot — 0 syscalls after initial submit |
| Read data | `read()`/`recv()` — 1 syscall per read | Batched, multishot — amortized ~0 |
| Per-loop syscalls | O(n) where n = events | O(1) or O(0) with SQPOLL |
