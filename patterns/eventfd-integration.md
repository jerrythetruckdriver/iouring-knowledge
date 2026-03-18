# Eventfd Integration

Bridging io_uring completions to external event loops.

## The Problem

io_uring has its own completion model (poll CQ ring or `io_uring_enter`). Most applications already have an event loop (epoll, libuv, GLib, Qt, etc.). Eventfd bridges the gap — kernel writes to eventfd on CQE posting, your event loop wakes up via its normal fd-watching mechanism.

## Registration

```c
int efd = eventfd(0, EFD_CLOEXEC | EFD_NONBLOCK);

// Notify on every CQE
io_uring_register_eventfd(ring, efd);

// Or: notify only for async completions (skip inline)
io_uring_register_eventfd_async(ring, efd);
```

`IORING_REGISTER_EVENTFD` (opcode 4) — notify on all CQEs.
`IORING_REGISTER_EVENTFD_ASYNC` (opcode 7) — notify only when completion happens asynchronously (io-wq, poll, etc.). Skips notification for inline completions where the submitter will see them immediately.

## Disable/Re-enable

```c
// Temporarily disable without unregistering
// Set IORING_CQ_EVENTFD_DISABLED in cq_ring->flags
*cq_flags |= IORING_CQ_EVENTFD_DISABLED;

// Re-enable
*cq_flags &= ~IORING_CQ_EVENTFD_DISABLED;
```

Useful during batch processing — disable notifications while draining CQ, re-enable before going back to sleep. Avoids thundering herd of eventfd writes during bulk completions.

## Unregister

```c
io_uring_unregister_eventfd(ring);  // opcode 5
```

## Pattern: Epoll + io_uring

```c
int efd = eventfd(0, EFD_NONBLOCK);
io_uring_register_eventfd(&ring, efd);

struct epoll_event ev = { .events = EPOLLIN, .data.fd = efd };
epoll_ctl(epoll_fd, EPOLL_CTL_ADD, efd, &ev);

while (1) {
    int n = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
    for (int i = 0; i < n; i++) {
        if (events[i].data.fd == efd) {
            // Drain eventfd
            uint64_t val;
            read(efd, &val, sizeof(val));
            // Process io_uring completions
            drain_cqes(&ring);
        }
        // ... handle other epoll fds
    }
}
```

This is the migration path. Keep your epoll loop for fds you haven't moved to io_uring yet, use eventfd for io_uring notifications. Gradually move fds to io_uring.

## Pattern: libuv Integration

```c
uv_poll_t poll_handle;
uv_poll_init(loop, &poll_handle, efd);
uv_poll_start(&poll_handle, UV_READABLE, on_uring_ready);

void on_uring_ready(uv_poll_t* handle, int status, int events) {
    uint64_t val;
    read(efd, &val, sizeof(val));
    drain_cqes(&ring);
}
```

Same pattern works for GLib (`g_io_add_watch`), Qt (`QSocketNotifier`), or any event loop that watches fds.

## When to Use Eventfd

**Use eventfd when:**
- Integrating io_uring into an existing event loop
- Running io_uring alongside epoll during migration
- Framework doesn't support io_uring natively
- Need to wake a thread that's blocked in a different event loop

**Don't use eventfd when:**
- Pure io_uring application — just call `io_uring_wait_cqe` or use registered wait
- Using SQPOLL — completions are produced without submission
- Using `IORING_OP_MSG_RING` for cross-ring signaling (lower overhead)
- Using `IORING_REGISTER_SEND_MSG_RING` for cross-thread wake (no fd needed)

## Eventfd vs MSG_RING vs SEND_MSG_RING

| Mechanism | Requires fd | Cross-ring | Cross-thread (no ring) | Overhead |
|-----------|-------------|------------|----------------------|----------|
| Eventfd | Yes | No (external) | Yes | write syscall |
| MSG_RING | No (uses ring) | Yes | No (need ring fd) | SQE submission |
| SEND_MSG_RING | No | Yes | Yes (6.10) | Register call |

MSG_RING and SEND_MSG_RING are io_uring-native. Use them when both sides are io_uring. Eventfd is for bridging to non-io_uring event loops.

## EVENTFD_ASYNC Optimization

If your loop already checks CQ after every submission batch:

```c
io_uring_register_eventfd_async(&ring, efd);
```

This skips eventfd notification for completions that happen synchronously during `io_uring_enter`. You'll catch those when you check CQ after submit returns. Only truly async completions (returned later via poll/io-wq) trigger the eventfd. Reduces write() overhead significantly in high-throughput scenarios.

## Gotchas

1. **Drain the eventfd**: Always `read()` the eventfd value before processing CQEs. Otherwise epoll won't re-trigger (edge-triggered) or you'll spin (level-triggered with stale data).
2. **Batch CQE processing**: One eventfd write may correspond to multiple CQEs. Always drain the entire CQ ring.
3. **EFD_SEMAPHORE**: Don't use it. You want to know "completions available", not count them individually.
4. **Thread safety**: Eventfd read/write is thread-safe. CQ ring access is not (unless you handle your own locking).
