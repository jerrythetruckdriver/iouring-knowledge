# Poll Operations

## IORING_OP_POLL_ADD (opcode 6)

Async poll(2). Watches a file descriptor for events. Available since 5.1.

### SQE Layout

```c
sqe->opcode      = IORING_OP_POLL_ADD;
sqe->fd          = target_fd;
sqe->poll32_events = POLLIN | POLLOUT | ...;  // standard poll events
sqe->len         = flags;  // POLL_ADD flags (not poll events!)
```

Note the split: **events** go in `poll32_events`, **control flags** go in `len`.

### Control Flags (sqe->len)

| Flag | Value | Since | Behavior |
|------|-------|-------|----------|
| `IORING_POLL_ADD_MULTI` | 1<<0 | 5.13 | Multishot: keep polling after each event |
| `IORING_POLL_UPDATE_EVENTS` | 1<<1 | 5.13 | Update events on existing poll |
| `IORING_POLL_UPDATE_USER_DATA` | 1<<2 | 5.13 | Update user_data on existing poll |
| `IORING_POLL_ADD_LEVEL` | 1<<3 | 5.18 | Level-triggered (force) |

### CQE Result

`cqe->res` contains the triggered poll event mask.

With `POLL_ADD_MULTI`:
- `IORING_CQE_F_MORE` set → more events coming, keep consuming
- `IORING_CQE_F_MORE` not set → poll terminated, resubmit if needed

### Trigger Semantics

**Without POLL_ADD_MULTI (one-shot):**
- Level-triggered on submission. If data is already ready, CQE fires immediately.

**With POLL_ADD_MULTI:**
- First completion: level-triggered (fires if ready now)
- Subsequent completions: **edge-triggered** (fires on state change only)

This is a subtle gotcha. Your first multishot CQE tells you "data is ready now." After that, you only get notified on *new* events.

### IORING_OP_POLL_REMOVE (opcode 7)

Removes or updates an existing poll request.

```c
sqe->opcode = IORING_OP_POLL_REMOVE;
sqe->addr   = original_user_data;  // identifies the poll to modify
```

**Update events:**
```c
sqe->opcode       = IORING_OP_POLL_REMOVE;
sqe->addr         = original_user_data;
sqe->len          = IORING_POLL_UPDATE_EVENTS;
sqe->poll32_events = new_events;
```

**Update user_data:**
```c
sqe->len = IORING_POLL_UPDATE_EVENTS | IORING_POLL_UPDATE_USER_DATA;
sqe->off = new_user_data;
```

### Patterns

**Multishot socket readability:**
```c
io_uring_prep_poll_multishot(sqe, sock_fd, POLLIN);
// CQEs arrive whenever socket becomes readable
// Edge-triggered after first — drain fully on each notification
```

**One-shot write readiness:**
```c
io_uring_prep_poll_add(sqe, sock_fd, POLLOUT);
// One CQE when socket is writable, then done
```

**Dynamic event update (add POLLOUT to existing POLLIN poll):**
```c
sqe->opcode       = IORING_OP_POLL_REMOVE;
sqe->addr         = original_user_data;
sqe->len          = IORING_POLL_UPDATE_EVENTS;
sqe->poll32_events = POLLIN | POLLOUT;
```

### Why Not Just Use Multishot Recv?

For most networking, `RECV` with `IORING_RECV_MULTISHOT` is better than POLL_ADD + separate RECV. But POLL_ADD still wins when:

- You need poll on non-socket fds (pipes, eventfds, device files)
- You want to watch for specific conditions (POLLHUP, POLLERR) without reading
- You're bridging io_uring into an existing event loop
- You need write readiness notification

### vs epoll

| | epoll | POLL_ADD |
|---|---|---|
| Syscalls | epoll_ctl + epoll_wait | One SQE |
| Multishot | EPOLLET is edge-triggered | First level, then edge |
| Batching | Separate add/modify | SQE batched with other ops |
| fd limit | Requires epoll fd management | Per-request, no extra fd |

POLL_ADD is io_uring's internal poll mechanism. When you submit a recv and the socket isn't ready, io_uring internally arms poll exactly like this. The explicit opcode gives you direct control over the same machinery.
