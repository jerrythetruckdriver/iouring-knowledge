# pidfd + io_uring

There is no `IORING_OP_PIDFD_OPEN`. But pidfd and io_uring interact in useful ways.

## What Works

### POLL_ADD on pidfd

pidfd file descriptors are pollable. When the process exits, the pidfd becomes readable. Multishot POLL_ADD gives you async process exit notification with zero syscalls:

```c
int pidfd = pidfd_open(child_pid, 0);

struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_poll_add(sqe, pidfd, POLLIN);
// No need for multishot — process exits once
io_uring_sqe_set_data64(sqe, PIDFD_EXIT_TAG);
```

CQE fires when the process exits. Then call `waitid(P_PIDFD, pidfd, ...)` to reap — or use `IORING_OP_WAITID`.

### WAITID as the Better pidfd_wait

`IORING_OP_WAITID` (6.7) is the async equivalent of waitid(2). It uses FUTEX2 flags and supports `P_PIDFD`:

```c
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_waitid(sqe, P_PIDFD, pidfd, &siginfo, WEXITED, 0);
```

This is more useful than POLL_ADD on pidfd because WAITID gives you the exit status directly in the siginfo struct. POLL_ADD just tells you "something happened."

### pidfd_send_signal via URING_CMD?

No. pidfd operations (pidfd_send_signal, pidfd_getfd) don't have URING_CMD support. pidfd is a pseudo-filesystem, not a character device with ioctl handlers.

### CLOSE on pidfd

`IORING_OP_CLOSE` works on pidfds. Clean up async:

```c
io_uring_prep_close(sqe, pidfd);
```

## Process Management Pattern

Async process pool with io_uring:

```c
// Spawn N workers
for (int i = 0; i < N; i++) {
    pid_t pid = fork();
    if (pid == 0) { exec_worker(); }
    
    int pidfd = pidfd_open(pid, 0);
    
    // Async wait for each
    struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
    io_uring_prep_waitid(sqe, P_PIDFD, pidfd, &siginfos[i], WEXITED, 0);
    io_uring_sqe_set_data64(sqe, WORKER_TAG | i);
}

io_uring_submit(&ring);

// Event loop — completions arrive as workers exit
while (active_workers > 0) {
    struct io_uring_cqe *cqe;
    io_uring_wait_cqe(&ring, &cqe);
    
    int worker_id = cqe->user_data & 0xFFFF;
    int exit_status = siginfos[worker_id].si_status;
    // Respawn, log, or handle
    
    io_uring_cqe_seen(&ring, cqe);
    active_workers--;
}
```

## vs Alternatives

| Method | Async? | Exit Status? | Race-free? | Syscalls |
|--------|--------|-------------|------------|----------|
| SIGCHLD handler | Yes | Via waitpid | Racy (signal coalescing) | 1+ |
| waitpid in thread | Blocking | Yes | Yes | 1 per wait |
| epoll + pidfd | Yes | Need waitid after | Yes | 2 |
| io_uring POLL_ADD + pidfd | Yes | Need waitid after | Yes | 0 (batched) |
| io_uring WAITID + pidfd | Yes | Yes (in siginfo) | Yes | 0 (batched) |

`IORING_OP_WAITID` with pidfd is the cleanest solution. One SQE, zero syscalls, exit status delivered directly.

## What's Missing

- No `IORING_OP_PIDFD_OPEN` — pidfd_open is one-time setup, not worth an opcode
- No `IORING_OP_PIDFD_SEND_SIGNAL` — could be useful for async kill, but not proposed
- No `IORING_OP_PIDFD_GETFD` — fd stealing is rare and security-sensitive

The combination of pidfd_open (sync, once) + WAITID (async, io_uring) covers the primary use case. The rest is niche enough that nobody's proposed opcodes for it.
