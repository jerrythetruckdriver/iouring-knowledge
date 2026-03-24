# Multi-Ring Event Loop Patterns

## Why Multiple Rings

A single ring handles everything, but separate rings let you:
- Isolate storage I/O from network I/O (different latency profiles)
- Use IOPOLL for storage without affecting network ops
- Size rings independently (storage: small + deep, network: large + shallow)
- Apply different setup flags per workload

## Architecture Patterns

### Pattern 1: Storage + Network Split

```
Storage Ring:
  IORING_SETUP_IOPOLL | IORING_SETUP_SINGLE_ISSUER | IORING_SETUP_NO_SQARRAY
  - O_DIRECT reads/writes with registered buffers
  - Linked write+fsync chains
  - IOPOLL for NVMe completion

Network Ring:
  IORING_SETUP_COOP_TASKRUN | IORING_SETUP_SINGLE_ISSUER | IORING_SETUP_DEFER_TASKRUN
  - Multishot accept + recv
  - Provided buffer rings
  - Send/send_zc
  - Timeouts
```

### Pattern 2: Control + Data Plane

```
Control Ring (small, 32 entries):
  - Accept connections
  - Timers and timeouts
  - MSG_RING dispatching
  - Cancel operations

Data Ring (large, 4096 entries):
  - All recv/send operations
  - Splice/sendfile
  - Buffer management
```

This is roughly what libuv does: `ctl` ring for epoll_ctl replacements, `iou` ring for file I/O.

### Pattern 3: Per-Priority Rings

```
High-Priority Ring:
  IORING_SETUP_SQPOLL (dedicated kernel thread)
  - Latency-sensitive control messages
  - Health checks, heartbeats
  - Admin operations

Bulk Ring:
  IORING_SETUP_COOP_TASKRUN
  - Bulk data transfer
  - Log writes
  - Background prefetch
```

## Unified Event Loop

The challenge: you need to wait on multiple rings simultaneously.

### Approach 1: eventfd Bridge

Register an eventfd with each ring. epoll/poll on all eventfds:

```c
int efd_storage = eventfd(0, EFD_NONBLOCK);
int efd_network = eventfd(0, EFD_NONBLOCK);
io_uring_register_eventfd(&storage_ring, efd_storage);
io_uring_register_eventfd(&network_ring, efd_network);

// Main loop: epoll on both eventfds
struct epoll_event events[2];
epoll_ctl(epfd, EPOLL_CTL_ADD, efd_storage, &ev_storage);
epoll_ctl(epfd, EPOLL_CTL_ADD, efd_network, &ev_network);

while (1) {
    int n = epoll_wait(epfd, events, 2, -1);
    for (int i = 0; i < n; i++) {
        if (events[i].data.fd == efd_storage)
            drain_cqes(&storage_ring);
        else
            drain_cqes(&network_ring);
    }
}
```

**Downside**: adds syscall overhead (epoll_wait + eventfd reads). Use `IORING_REGISTER_EVENTFD_ASYNC` to suppress eventfd for inline completions.

### Approach 2: Alternating Poll

No eventfd. Round-robin check both rings with `io_uring_peek_cqe()`:

```c
while (1) {
    // Non-blocking check both rings
    io_uring_peek_cqe(&storage_ring, &cqe);
    if (cqe) process_storage(cqe);

    io_uring_peek_cqe(&network_ring, &cqe);
    if (cqe) process_network(cqe);

    // If both empty, block on one with timeout
    struct __kernel_timespec ts = { .tv_nsec = 100000 }; // 100µs
    io_uring_wait_cqe_timeout(&network_ring, &cqe, &ts);
}
```

**Downside**: busy-loop or latency tradeoff. Works well with SQPOLL (kernel keeps producing CQEs).

### Approach 3: POLL_ADD on Ring FD

Use one ring to poll another ring's fd:

```c
// Network ring polls storage ring's fd for readability (CQEs available)
io_uring_prep_poll_add(sqe, storage_ring.ring_fd, POLLIN);
sqe->flags |= IOSQE_IO_LINK; // chain with processing
```

Clever but fragile. Not widely used.

### Approach 4: MSG_RING Notification

Storage ring sends MSG_RING to network ring when batch completes:

```c
// In storage completion handler:
io_uring_prep_msg_ring(sqe, network_ring.ring_fd,
                       STORAGE_BATCH_DONE, batch_id, 0);
```

Network ring's event loop sees a CQE with `STORAGE_BATCH_DONE` result. No eventfd needed. Low overhead.

## Shared Resources Between Rings

### Clone Buffers

```c
struct io_uring_clone_buffers cb = {
    .src_fd = storage_ring.ring_fd,
    .flags = IORING_REGISTER_SRC_REGISTERED,
};
io_uring_register(network_ring.ring_fd, IORING_REGISTER_CLONE_BUFFERS, &cb, 1);
```

Both rings share the same pinned buffer pages. No duplication.

### Shared Fixed Files

No direct sharing. Each ring has its own fixed file table. Workaround: register the same fds in both rings, or use MSG_RING with `IORING_MSG_SEND_FD` to pass fixed descriptors.

## When to Use Multiple Rings

| Scenario | Single Ring | Multiple Rings |
|----------|-------------|----------------|
| Simple server (accept+recv+send) | ✅ | Overkill |
| Database (network + O_DIRECT storage) | Possible | ✅ Better isolation |
| Proxy (network only, two directions) | ✅ | Unnecessary |
| Mixed IOPOLL + non-IOPOLL | ❌ Can't mix | ✅ Required |
| Different latency SLOs | Difficult | ✅ Per-ring tuning |
| Thread-per-core | One per thread | One per thread is fine |

## Anti-Patterns

- **Ring per connection**: Too many rings. Use one ring per thread.
- **Shared ring without SQPOLL**: Multiple threads submitting to one ring without SQPOLL requires external locking — defeats the purpose.
- **eventfd per ring + epoll**: If you're using epoll to multiplex rings, you've added a syscall layer. Consider whether one ring would suffice.
