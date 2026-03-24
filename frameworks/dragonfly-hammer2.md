# DragonflyBSD HAMMER2 and io_uring

## Short Answer

HAMMER2 has nothing to do with io_uring. They're on different operating systems.

## HAMMER2

HAMMER2 is DragonflyBSD's default filesystem. Block copy-on-write, instant snapshots, compression, dedup. Well-designed for its platform.

DragonflyBSD uses kqueue for event notification. No io_uring, no plans for io_uring. It's a BSD — they have their own kernel architecture.

## DragonflyBSD I/O Stack

- **Event notification**: kqueue
- **Async I/O**: POSIX AIO (aio_read/aio_write)
- **Block layer**: Standard BSD CAM/GEOM-like stack
- **No completion-based I/O**: Same gap as FreeBSD/OpenBSD

## Why This Comes Up

People conflate "advanced filesystem" with "advanced I/O interface." They're orthogonal:

- HAMMER2 is about **data organization** (B-tree, copy-on-write, snapshots)
- io_uring is about **I/O submission** (SQ/CQ, batching, zero-copy)

A filesystem benefits from io_uring when applications access it via io_uring operations. But HAMMER2 runs on DragonflyBSD, which doesn't have io_uring.

## Cross-Pollination

The only intersection is conceptual: both io_uring and HAMMER2 think carefully about reducing unnecessary work. HAMMER2's radix tree allows skipping levels for small files. io_uring's batched submission eliminates per-operation syscall overhead. Different problems, same philosophy.

## See Also

- `pitfalls/bsd-portability.md` — BSD I/O models vs io_uring
