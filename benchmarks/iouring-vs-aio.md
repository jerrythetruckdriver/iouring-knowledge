# io_uring vs aio

## aio is Dead

Linux AIO (`libaio`, `io_submit`, `io_getevents`) was the previous async I/O API. It had fundamental problems:

1. **O_DIRECT only** — buffered I/O falls back to sync. Useless for most applications.
2. **No networking** — file I/O only. Need epoll separately for sockets.
3. **Limited operations** — read/write/fsync. That's it.
4. **No buffer registration** — every operation maps/unmaps user pages.
5. **Per-operation syscall overhead** — `io_submit()` does one syscall per batch, but no batched completion wait.
6. **No linking** — can't chain dependent operations.

## Performance

From Jens Axboe's io_uring paper, 4K random reads on NVMe:

| Interface | IOPS | Relative |
|-----------|------|----------|
| aio | 608K | 1.0x |
| io_uring | 1.2M | 2.0x |
| io_uring (polled) | 1.7M | 2.8x |

io_uring is 2x faster in the same mode, 2.8x with polling. And aio doesn't even have a polled mode.

## Feature Comparison

| Feature | aio | io_uring |
|---------|-----|----------|
| Buffered I/O | No (falls back to sync) | Yes |
| Networking | No | Yes |
| Opcodes | ~3 | 65+ |
| Buffer registration | No | Yes |
| File registration | No | Yes |
| Linked operations | No | Yes |
| Multishot | No | Yes |
| Zero-copy | No | Yes (send + recv) |
| SQPOLL | No | Yes |
| Kernel poll | No | Yes (IOPOLL) |

## Migration

If you're still using aio in 2026, you're leaving performance on the table. io_uring is a strict superset. The only reason to keep aio is kernel version constraints (pre-5.1).
