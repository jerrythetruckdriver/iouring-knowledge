# inotify/fanotify + io_uring Integration

There is no `IORING_OP_INOTIFY_ADD_WATCH` or `IORING_OP_FANOTIFY_MARK`. io_uring doesn't replace filesystem notification APIs — it bridges to them.

## How It Works

inotify and fanotify both produce file descriptors. Those fds are pollable. io_uring reads from pollable fds just fine.

### Pattern: POLL_ADD on inotify fd

```c
int ifd = inotify_init1(IN_NONBLOCK);
inotify_add_watch(ifd, "/path", IN_MODIFY | IN_CREATE | IN_DELETE);

// Multishot poll — get notified every time events are available
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_poll_add(sqe, ifd, POLLIN);
sqe->len = IORING_POLL_ADD_MULTI;  // multishot
io_uring_sqe_set_data64(sqe, INOTIFY_POLL_TAG);
```

When CQE fires with POLLIN, read from the inotify fd:

```c
// Can use io_uring READ or just a synchronous read — events are ready
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, ifd, buf, sizeof(buf), 0);
```

### Pattern: Multishot READ on inotify fd

Since inotify fds support read(2), you can use `IORING_OP_READ_MULTISHOT` (6.7+) directly:

```c
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_read_multishot(sqe, ifd, 0, 0, /* bgid */ 1);
```

Each CQE delivers a buffer with one or more `struct inotify_event`. Parse them sequentially (variable-length due to name field).

### Pattern: fanotify with FAN_REPORT_FID

fanotify fds work identically — they're pollable, readable. Use POLL_ADD or READ:

```c
int fan_fd = fanotify_init(FAN_CLASS_NOTIF | FAN_REPORT_FID, O_RDONLY);
fanotify_mark(fan_fd, FAN_MARK_ADD | FAN_MARK_FILESYSTEM,
              FAN_CREATE | FAN_DELETE | FAN_MODIFY, AT_FDCWD, "/mnt");

// Same poll pattern as inotify
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_poll_add(sqe, fan_fd, POLLIN);
sqe->len = IORING_POLL_ADD_MULTI;
```

## What Would a Native Op Look Like?

An `IORING_OP_INOTIFY_ADD_WATCH` would be pointless — watch setup is a one-time operation. The ongoing work (event delivery) already works through io_uring via poll/read.

What might actually help:
- `IORING_OP_FANOTIFY_MARK` — for dynamic mark/unmark in a hot path (container orchestration, CI builds)
- Direct event delivery without read parsing — but this would require kernel changes to inotify/fanotify subsystems

Neither is planned. The current approach works.

## Comparison

| Approach | Syscalls | Latency | Complexity |
|----------|----------|---------|------------|
| epoll + inotify | 2 (epoll_wait + read) | ~2-5μs | Low |
| io_uring POLL_ADD + READ | 0 (batched) | ~1-3μs | Medium |
| io_uring READ_MULTISHOT | 0 (zero syscall) | ~1-2μs | Medium |
| Blocking read in thread | 1 (read) | Wake latency | Low |

## Practical Considerations

- **inotify events are small** (typically 16-272 bytes). The I/O overhead is negligible. io_uring helps mainly by eliminating the poll syscall, not by accelerating the read.
- **Watch limits** (`/proc/sys/fs/inotify/max_user_watches`) are the real bottleneck, not event delivery speed.
- **fanotify permission events** (FAN_OPEN_PERM) require writing a response back. Can do that via io_uring WRITE to the fanotify fd.
- **Neither API has an `IORING_OP_INOTIFY_INIT` or `IORING_OP_FANOTIFY_INIT`** — setup is always synchronous. Fine — you do it once.

## When io_uring Actually Matters for Notifications

When you're already running an io_uring event loop for networking/storage and want to integrate filesystem watches without a separate epoll fd or thread. Single event loop, single completion path. That's the value — architectural simplicity, not raw performance.
