# Connection Migration Between Rings/Threads

## The Problem

Thread-per-core architecture: each thread owns a ring. A connection accepted on thread A needs to move to thread B for load balancing or affinity.

## Three Approaches

### 1. MSG_RING with fd Passing (Recommended)

```c
// Thread A: accept connection, send fd to thread B's ring
int client_fd = accept(...);  // or from multishot accept CQE

// Pass fd to thread B's ring
io_uring_prep_msg_ring_fd(sqe, ring_b_fd, client_fd,
                           target_fixed_slot, user_data, 0);
// Thread B gets a CQE with the fd installed in its fixed file table
```

**Pros:** Zero-copy fd transfer, CQE notification, works with fixed files.
**Cons:** Requires IORING_MSG_SEND_FD (6.0+), target ring must have free fixed file slots.

### 2. REGISTER_SEND_MSG_RING (Ringless)

For worker threads that don't own a ring:

```c
// Worker thread (no ring): signal the target ring
struct io_uring_sqe sqe = {
    .opcode = IORING_OP_MSG_RING,
    .fd = target_ring_fd,
    ...
};
io_uring_register(target_ring_fd, IORING_REGISTER_SEND_MSG_RING, &sqe, 1);
// Target ring gets a CQE — data only, no fd passing
```

Worker passes a message; the fd must be shared via SCM_RIGHTS or installed separately.

### 3. Direct Descriptor Install

Connection stays on original ring; install the fd into the target ring's fixed file table:

```c
// Thread A owns the connection
// Install fd into thread B's table via FIXED_FD_INSTALL
io_uring_prep_fixed_fd_install(sqe, fixed_slot, 0);
// Now both rings can reference the fd
// But only one should do I/O at a time!
```

**Warning:** Two rings doing I/O on the same fd = race condition. Use this for hand-off, not sharing.

## Migration Patterns

### Accept → Dispatch (Load Balancer)

```c
// Acceptor thread: one ring with multishot accept
io_uring_prep_multishot_accept_direct(sqe, listen_fd, ...);

// On each accept CQE:
int target = next_thread();  // round-robin, least-connections, etc.
io_uring_prep_msg_ring_fd(sqe, rings[target],
    accepted_fd, IORING_FILE_INDEX_ALLOC, encode_new_conn, 0);
```

### Work Stealing

```c
// Idle thread B steals from busy thread A
// Thread A: post notification to shared steal queue
// Thread B: pick up fd from steal queue
// Thread B: MSG_RING_FD to install in own ring
// Thread A: cancel any pending ops on that fd first!
```

**Critical:** Cancel all in-flight operations on a connection BEFORE migrating it. Completing a recv on thread A while thread B starts sending = corruption.

### Graceful Migration Sequence

```c
// 1. Thread A: cancel all pending ops on the fd
io_uring_prep_cancel(sqe, old_user_data, IORING_ASYNC_CANCEL_ALL);

// 2. Wait for all cancellation CQEs

// 3. Thread A: send fd to Thread B via MSG_RING
io_uring_prep_msg_ring_fd(sqe, ring_b_fd, client_fd, slot, data, 0);

// 4. Thread B: CQE arrives, start I/O on the connection
io_uring_prep_recv_multishot(sqe, slot, ...);
sqe->flags |= IOSQE_FIXED_FILE;
```

## What You Can't Migrate

- **Registered buffers** — per-ring. Target ring needs its own or clone.
- **Provided buffer rings** — per-ring. Target needs its own buffer groups.
- **In-flight operations** — must complete or cancel before migration.
- **Multishot subscriptions** — terminate on the source ring, re-arm on target.
- **SQPOLL state** — SQPOLL thread is per-ring.

## Comparison with Alternatives

| Method | Overhead | fd Transfer | Notification | Kernel Version |
|--------|----------|-------------|--------------|----------------|
| MSG_RING fd | Low | Yes | CQE | 6.0 |
| SCM_RIGHTS | Medium | Yes | None (need signaling) | Any |
| dup2 + signal | High | Copy, not transfer | Signal/eventfd | Any |
| Shared fd (no migration) | Zero | N/A | N/A | Any |

## When Not to Migrate

If your load is roughly balanced, accepting on one thread and processing on another adds latency and complexity. Consider:
- **SO_REUSEPORT** — kernel distributes connections to per-thread listeners
- **RSS steering** — NIC hardware distributes by flow hash
- Both avoid migration entirely.

## See Also

- [Ring Messaging](ring-messaging.md) — MSG_RING and SEND_MSG_RING details
- [Thread-Per-Core](thread-per-core.md) — architecture patterns
- [SO_REUSEPORT + Accept](reuseport-accept.md) — avoiding migration entirely
- [Cancel Patterns](cancel-patterns.md) — canceling before migration
