# io_uring and Signals

Signals and io_uring interact in three ways: signal masking during waits, SIGCHLD replacement via waitid, and signal-safe operation submission.

## Signal Masking

`io_uring_enter()` with `IORING_ENTER_EXT_ARG` accepts a signal mask:

```c
struct io_uring_getevents_arg arg = {
    .sigmask = (unsigned long)&sigmask,
    .sigmask_sz = sizeof(sigset_t),
    .ts = (unsigned long)&ts,
};
io_uring_enter(ring_fd, 0, 1, IORING_ENTER_GETEVENTS | IORING_ENTER_EXT_ARG, &arg);
```

This is equivalent to `pselect` / `ppoll` / `epoll_pwait`: atomically set the signal mask, wait for events, restore the signal mask. Prevents the race between signal delivery and blocking wait.

With registered wait (`io_uring_reg_wait`, 6.13+):

```c
struct io_uring_reg_wait rw = {
    .ts = { .tv_sec = 1 },
    .sigmask = (unsigned long)&sigmask,
    .sigmask_sz = sizeof(sigset_t),
};
/* Submit with registered wait index */
```

Same semantics, but the wait parameters are in pre-registered memory. No copy from userspace on every wait call.

## Replacing SIGCHLD with IORING_OP_WAITID

Traditional child process management:
1. Set up `SIGCHLD` handler or use `signalfd`
2. `waitpid()` in handler or event loop

io_uring way (6.7+):
```c
sqe = io_uring_get_sqe(&ring);
io_uring_prep_waitid(sqe, P_ALL, 0, &info, WEXITED, 0);
```

No signal handler. No signalfd. Child exit is a CQE like any other completion. See [waitid.md](waitid.md).

## Replacing signalfd

signalfd converts signals to readable events on a file descriptor. Works with epoll, works with io_uring (POLL_ADD on a signalfd).

But with io_uring, you can often replace what signals do:
- **SIGCHLD** → `IORING_OP_WAITID`
- **SIGALRM / timers** → `IORING_OP_TIMEOUT`
- **SIGUSR1/2 (custom wakeup)** → `IORING_OP_MSG_RING` or `IORING_REGISTER_SEND_MSG_RING`
- **SIGPIPE** → Already irrelevant (ignored in most modern code)
- **SIGTERM/SIGINT** → Still need signalfd or signal handler (process lifecycle)

Pattern: use signalfd only for process lifecycle signals (SIGTERM, SIGINT, SIGHUP). Route it through `IORING_OP_POLL_ADD` multishot:

```c
int sfd = signalfd(-1, &sigmask, SFD_NONBLOCK);
sqe = io_uring_get_sqe(&ring);
io_uring_prep_poll_multishot(sqe, sfd, POLLIN);
```

Now signal delivery is a CQE. Read the `struct signalfd_siginfo` from the signalfd when you get the poll event.

## Signal Safety in io_uring Operations

io_uring SQE submission is not async-signal-safe. Do not call `io_uring_submit()` or manipulate the SQ ring from a signal handler.

However, you can safely notify the ring from a signal handler using:
1. **eventfd** — `write()` to a registered eventfd is async-signal-safe
2. **`IORING_REGISTER_SEND_MSG_RING`** — NOT safe from signal handlers (requires `io_uring_register` syscall)

Best practice: block all signals in io_uring threads, handle them via signalfd.

## EINTR Handling

io_uring operations that go async never return `-EINTR`. The request is queued in the kernel and completes regardless of signal delivery.

`io_uring_enter()` for waiting CAN return `-EINTR` if a signal is delivered while waiting. Use `IORING_ENTER_EXT_ARG` with a signal mask to control this.

liburing's `io_uring_wait_cqe()` retries on `-EINTR` internally. If you're using raw syscalls, handle the retry yourself.
