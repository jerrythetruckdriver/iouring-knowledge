# IORING_OP_WAITID — Async Process Waiting

Opcode 48. Added in kernel 6.7. Async version of `waitid(2)`.

## Why It Matters

Traditional `waitid()` blocks until a child changes state. In an event-driven architecture, blocking to reap children is embarrassing. `IORING_OP_WAITID` makes process management a first-class async citizen.

## SQE Layout

```
sqe->fd        = P_PID / P_PGID / P_PIDFD / P_ALL  (idtype)
sqe->len       = pid                                 (id)
sqe->file_index = wait options (WEXITED, WSTOPPED, etc.)
sqe->addr2     = pointer to siginfo_t (filled on completion)
sqe->waitid_flags = 0 (reserved)
```

## CQE Result

- `res >= 0` — success, siginfo_t populated
- `res < 0` — negated errno

## Use Cases

1. **Process supervisors** — monitor child processes without dedicated reaper threads
2. **Build systems** — track parallel compilation jobs through the ring
3. **Container runtimes** — reap init process children asynchronously

## Pattern: Async Process Pool

```c
// Submit waitid for each spawned child
for (int i = 0; i < nchildren; i++) {
    struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
    io_uring_prep_waitid(sqe, P_PID, pids[i],
                         &infos[i], WEXITED, 0);
    sqe->user_data = pids[i];
}
io_uring_submit(&ring);

// Event loop picks up child exits alongside I/O completions
// No SIGCHLD handler. No waitpid loop. Just CQEs.
```

## Comparison with Alternatives

| Method | Blocking? | Batched? | Works with io_uring event loop? |
|--------|-----------|----------|---------------------------------|
| `waitpid()` | Yes | No | No |
| `waitid()` | Yes | No | No |
| `SIGCHLD` handler | Signal-based | No | Awkward |
| `pidfd_open()` + epoll | No | Via epoll | Separate loop |
| `IORING_OP_WAITID` | No | Yes | Native |

## Constraints

- The siginfo_t buffer must remain valid until the CQE arrives
- Cancelable via `IORING_OP_ASYNC_CANCEL`
- Follows same permission model as `waitid(2)`
