# liburing Release History

The userspace library for io_uring. Tracks kernel features, provides prep helpers, manages ring lifecycle.

## Version Timeline

| Version | Date | Key Changes |
|---------|------|-------------|
| 2.14 | Feb 2026 | Full API man page coverage, section 7 concept docs, test/build improvements |
| 2.13 | Dec 2025 | query.h install fix, const qualifiers, tsan builds, SQE lifetime docs, NO_MMAP fixes |
| 2.12 | Aug 2025 | Resize rings, zcrx support, registered wait, hybrid IOPOLL prep helpers |
| 2.11 | Jun 2025 | Bundle ops, vectorized send, mixed CQE/SQE support |
| 2.10 | Mar 2025 | Clone buffers, send-msg-ring registration, NAPI support |
| 2.9 | Dec 2024 | Futex ops, fixed fd install, ftruncate, NO_SQARRAY support |
| 2.8 | Oct 2024 | Bind/listen prep, multishot improvements |
| 2.7 | Jul 2024 | Memory region registration, clock registration |
| 2.6 | Mar 2024 | File alloc range, PBUF status, sync cancel |
| 2.5 | Sep 2023 | Send/recv ZC, CQE_SKIP_SUCCESS, msg_ring improvements |

## liburing 2.14 (Feb 2026) — Current

The entire liburing API is now documented with man pages. Section 7 pages cover io_uring concepts (ring setup, submission, completion, buffer management).

Notable changes:
- `io_uring_prep_epoll_wait()` — new prep helper for IORING_OP_EPOLL_WAIT
- `io_uring_prep_pipe()` — prep helper for IORING_OP_PIPE
- Net zero-copy benchmark updates
- Test improvements: epwait SQE/CQE counting, min-timeout CQE verification
- Build fixes: `io_uring_alloc_huge` ring memory calculation, NO_MMAP error paths

## liburing 2.13 (Dec 2025)

- `query.h` now installed (IORING_REGISTER_QUERY support)
- const qualifiers added to prep functions
- Thread sanitizer (tsan) enabled builds
- SQE pointer lifetime documentation in headers
- Fix: NO_MMAP munmap error paths, PA-RISC buf ring mmap

## API Philosophy

liburing wraps the raw io_uring syscall interface with:

1. **Prep helpers** — `io_uring_prep_*()` functions that fill SQE fields correctly
2. **Ring management** — `io_uring_queue_init()`, `io_uring_submit()`, `io_uring_peek_cqe()`
3. **Feature probing** — `io_uring_opcode_supported()`, or now `IORING_REGISTER_QUERY`
4. **Memory management** — huge page allocation, NO_MMAP support

## Building

```bash
./configure && make && make install
# Or just include liburing.h and link -luring
```

No external dependencies. Header-only usage possible for simple cases via inline prep helpers.

## Source

- Repository: https://github.com/axboe/liburing
- Man pages: `man 3 io_uring_prep_*`, `man 7 io_uring`
