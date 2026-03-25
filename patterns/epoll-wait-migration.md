# IORING_OP_EPOLL_WAIT Migration Guide

## The Op (6.15)

`IORING_OP_EPOLL_WAIT` (opcode 57) lets you submit an epoll_wait as an io_uring SQE. The epoll fd is monitored asynchronously — when events are ready, a CQE is posted with the number of ready events.

```c
struct epoll_event events[64];
io_uring_prep_epoll_wait(sqe, epoll_fd, events, 64, 0);
// CQE res = number of ready events (like epoll_wait return value)
```

## Why This Exists

Not to replace epoll with io_uring — to let you **incrementally** move from epoll to io_uring without rewriting everything at once.

### Migration Phases

**Phase 0: Pure epoll (starting point)**
```
epoll_wait() → dispatch events → read/write syscalls
// 1 + N syscalls per iteration
```

**Phase 1: epoll_wait via io_uring**
```
io_uring EPOLL_WAIT → dispatch events → read/write syscalls
// epoll_wait is now async, but still doing individual read/write syscalls
```

**Phase 2: epoll_wait + io_uring I/O**
```
io_uring EPOLL_WAIT → submit read/write via io_uring
// Both event notification and I/O through io_uring
// Still using epoll for readiness, but batching I/O
```

**Phase 3: Replace epoll for hot-path fds**
```
// Cold fds: EPOLL_WAIT (unchanged)
// Hot fds: multishot recv/send (pure io_uring)
// Gradual migration, one fd type at a time
```

**Phase 4: Pure io_uring**
```
// multishot accept → multishot recv → send/send_zc
// No epoll. All io_uring. Zero-syscall steady state with SQPOLL.
```

## SQE Layout

```c
struct io_uring_sqe {
    .opcode = IORING_OP_EPOLL_WAIT,  // 57
    .fd     = epoll_fd,
    .addr   = (uintptr_t)events,      // struct epoll_event *
    .len    = maxevents,               // max events to return
    // timeout via linked TIMEOUT, not built-in
};
```

## Key Differences from epoll_wait(2)

| Aspect | epoll_wait(2) | IORING_OP_EPOLL_WAIT |
|--------|---------------|---------------------|
| Blocking | Yes (with timeout) | No (async) |
| Timeout | Built-in parameter | Use linked IORING_OP_LINK_TIMEOUT |
| Signal mask | epoll_pwait2 | io_uring_enter EXT_ARG |
| Return value | Direct | Via CQE res |
| Batching | Standalone syscall | Part of io_uring submission batch |

## When to Use EPOLL_WAIT Op

**Good reasons:**
- Large codebase with epoll deeply embedded — phase 1 migration
- Mixed architecture: some subsystems on epoll, others on io_uring
- Third-party libraries that only expose epoll fds
- eventfd/timerfd/signalfd that you want in your io_uring event loop

**Bad reasons:**
- Greenfield project — just use io_uring natively
- Already using multishot recv — you don't need epoll anymore
- Performance-critical path — multishot recv beats epoll + recv

## Comparison with POLL_ADD

`POLL_ADD` monitors a single fd for readiness. `EPOLL_WAIT` monitors an epoll instance (which can watch thousands of fds). Different tools:

```c
// POLL_ADD: watch one fd
io_uring_prep_poll_multishot(sqe, client_fd, POLLIN);

// EPOLL_WAIT: watch an epoll set (many fds)
io_uring_prep_epoll_wait(sqe, epoll_fd, events, 64, 0);
```

For new connections: multishot recv is better than POLL_ADD → recv.
For legacy integration: EPOLL_WAIT bridges existing epoll infrastructure.

## Pattern: Hybrid Event Loop

```c
// One io_uring ring handles everything
while (1) {
    // Submit: EPOLL_WAIT for legacy fds + io_uring ops for new fds
    io_uring_prep_epoll_wait(sqe1, legacy_epfd, events, 64, 0);
    
    // Reap completions
    io_uring_submit_and_wait(ring, 1);
    
    for each CQE {
        if (is_epoll_completion) {
            // Dispatch legacy events
            for (int i = 0; i < cqe->res; i++)
                handle_legacy_event(&events[i]);
        } else {
            // Handle io_uring completions directly
            handle_iouring_completion(cqe);
        }
    }
}
```

## See Also

- [epoll Migration](epoll-migration.md) — broader epoll → io_uring migration strategy
- [EPOLL_CTL Op](epoll-ctl-op.md) — async epoll_ctl via io_uring
- [eventfd Integration](eventfd-integration.md) — bridging event loops
- [Multi-Ring Patterns](multi-ring-patterns.md) — unified event loop architectures
