# BSD and Other OS — No io_uring, Different Models

## FreeBSD

FreeBSD has no io_uring equivalent. Its async I/O stack:

- **kqueue** — Event notification (analogous to epoll, not io_uring). Edge-triggered, works on sockets, files, signals, processes, timers. More unified than epoll but still readiness-based.
- **POSIX AIO** — Async read/write via `aio_read()`/`aio_write()`. Implemented with kernel threads (`aiod`, `soaiod`). Limited to read/write/fsync — no networking, no linked operations, no batching.
- **sendfile()** — Zero-copy file→socket, more mature than Linux's version.

**What's missing vs io_uring:**
- No submission/completion queue model
- No syscall batching (each operation = 1 syscall)
- No linked operations or operation chaining
- No registered buffers/files
- No multishot operations
- No zero-copy receive
- No kernel polling (SQPOLL/IOPOLL equivalent)

**No plans to adopt io_uring.** FreeBSD's philosophy is to improve kqueue. The kqueue API is already more capable than epoll (unified event types), but it's fundamentally readiness-based, not completion-based.

## DragonflyBSD

Uses kqueue. Matthew Dillon has discussed completion-based I/O but no implementation exists. DragonflyBSD's focus is on its HAMMER2 filesystem and SMP scalability.

## OpenBSD

kqueue + pledge/unveil security model. io_uring's attack surface (kernel complexity, CVE history) is antithetical to OpenBSD's security-first philosophy. Will never adopt it.

## macOS / Darwin

kqueue inherited from FreeBSD. Apple has no public plans for completion-based I/O. Grand Central Dispatch (GCD) handles async I/O at the framework level but uses kqueue underneath.

## Windows

I/O Completion Ports (IOCP) — the original completion-based model. Predates io_uring by decades. io_uring took inspiration from IOCP's completion queue design. IOCP lacks: batched submission, registered resources, linked operations, zero-copy, kernel polling.

Registered I/O (RIO) in Windows Server 2012+ adds pre-registered buffers and batched completion — closer to io_uring but networking-only.

## Cross-Platform Implications

Frameworks targeting multiple OSes can't use io_uring exclusively:

| OS | Best Available | Completion-Based? |
|----|---------------|-------------------|
| Linux 5.1+ | io_uring | Yes |
| FreeBSD | kqueue + POSIX AIO | Partial |
| macOS | kqueue | No |
| Windows | IOCP / RIO | Yes |

This is why projects like **compio** (Rust) abstract over io_uring/IOCP/kqueue with a completion-based API, and why most frameworks default to the lowest common denominator (epoll/kqueue readiness model).

## The Reality

io_uring is a Linux-only advantage. If you need portability, you eat the performance cost. If you're Linux-only (databases, CDN edge nodes, container workloads), io_uring is the clear winner.

The BSDs won't catch up. They don't need to — their workloads are different, their kernels are smaller, and kqueue is good enough for what they do. The performance gap matters most at scale, and at scale, you're running Linux.
