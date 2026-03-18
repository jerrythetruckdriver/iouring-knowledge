# IORING_OP_EPOLL_CTL

Opcode 29 (enum value). Available since kernel 5.6.

Async `epoll_ctl(2)` — add, modify, or remove file descriptors from an epoll instance without a syscall.

## Why It Exists

For incremental migration. If your application uses epoll and you're moving to io_uring piece by piece, you can manage the epoll set from the io_uring submission path. One fewer syscall per epoll modification.

Also useful for the hybrid pattern: epoll for event notification, io_uring for I/O operations. `EPOLL_CTL` via SQE keeps the epoll set management on the submission queue.

## SQE Layout

```c
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_epoll_ctl(sqe, epfd, fd, op, ev);
```

| SQE field | Value |
|-----------|-------|
| `opcode` | `IORING_OP_EPOLL_CTL` (29) |
| `fd` | epoll fd |
| `addr` | pointer to `struct epoll_event` (for ADD/MOD) |
| `len` | `EPOLL_CTL_ADD`, `EPOLL_CTL_DEL`, or `EPOLL_CTL_MOD` |
| `off` | target fd to add/modify/remove |

## CQE Result

- `0` on success
- Negative errno on failure (same errors as `epoll_ctl(2)`: `-EEXIST`, `-ENOENT`, `-EBADF`, etc.)

## Pattern: Hybrid epoll + io_uring

```c
/* Use epoll for readiness notification */
/* Use io_uring for actual I/O and epoll management */

/* Add fd to epoll set via io_uring */
struct epoll_event ev = { .events = EPOLLIN, .data.fd = client_fd };
sqe = io_uring_get_sqe(&ring);
io_uring_prep_epoll_ctl(sqe, epfd, client_fd, EPOLL_CTL_ADD, &ev);

/* Link: then start multishot recv */
sqe->flags |= IOSQE_IO_LINK;
sqe = io_uring_get_sqe(&ring);
io_uring_prep_recv_multishot(sqe, client_fd, NULL, 0, 0);
sqe->flags |= IOSQE_BUFFER_SELECT;
sqe->buf_group = bgid;
```

## vs IORING_OP_EPOLL_WAIT (6.15)

`EPOLL_CTL` modifies the epoll set. `EPOLL_WAIT` (6.15) reads events from it.

Together, they make epoll fully manageable from io_uring:
- `EPOLL_CTL` (5.6) — add/mod/del
- `EPOLL_WAIT` (6.15) — wait for events

But at that point, you're using io_uring to drive epoll to do what io_uring can do natively with multishot recv/accept/poll. The main value is migration, not destination.

## When To Use

- Migrating from epoll incrementally
- Hybrid architectures where epoll handles some FDs
- Reducing syscalls in epoll-based code

## When Not To Use

- Greenfield io_uring applications (use multishot recv/accept directly)
- Performance-critical paths (native io_uring ops are more efficient than io_uring→epoll→io_uring)
