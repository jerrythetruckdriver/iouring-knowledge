# io_uring vs epoll

## The Fundamental Difference

epoll requires **2 syscalls per event**: `epoll_wait()` to get readiness, then `read()`/`write()`/`accept()` to do the I/O. That's a minimum of 2 user-kernel transitions per operation.

io_uring batches everything. One `io_uring_enter()` call submits N operations and waits for M completions. In SQPOLL mode, zero syscalls.

## Syscall Overhead

| Model | Syscalls per operation |
|-------|----------------------|
| epoll (single event) | 2 (epoll_wait + read/write) |
| epoll (batched) | 1 + N (epoll_wait + N reads) |
| io_uring (basic) | 1/N (amortized over batch) |
| io_uring (SQPOLL) | 0 |
| io_uring (DEFER_TASKRUN) | 1/N with minimal task work |

Post-Spectre/Meltdown, each syscall costs ~1-5µs depending on mitigation level. At 1M ops/sec, that's 1-5 seconds of CPU time per second just on transitions.

## Network Server Benchmarks

From various published benchmarks and framework comparisons:

### Connections/sec (accept throughput)
- epoll: ~100-200K accepts/sec (one accept per epoll_wait cycle)
- io_uring multishot accept: ~300-500K accepts/sec (one SQE, continuous CQEs)
- With fixed file allocation: higher still (no fd table manipulation)

### HTTP request throughput (small responses)
Typical numbers from echo/HTTP benchmark suites:
- epoll-based servers: 400K-800K req/sec (single core)
- io_uring-based servers: 600K-1.2M req/sec (single core)
- Improvement: 30-70% depending on workload

### Context switches
io_uring dramatically reduces context switches. A busy epoll server does 2x the context switches of an equivalent io_uring server under load.

## Storage I/O

From Jens Axboe's io_uring paper (4K random reads, NVMe):

| Interface | IOPS |
|-----------|------|
| aio | 608K |
| io_uring (non-polled) | 1.2M |
| io_uring (polled) | 1.7M |
| io_uring NOP (interface overhead) | 20M/sec |

## Where epoll Still Wins

- **Simplicity**: epoll is well-understood, battle-tested, works everywhere
- **Portability**: epoll works on older kernels. io_uring needs 5.1+ (realistically 5.19+ for modern features)
- **Compatibility**: Some features (TLS in kernel, etc.) aren't io_uring-aware yet
- **Small scale**: Below ~10K ops/sec, the overhead difference is noise

## Where io_uring Dominates

- **High connection rate**: Multishot accept crushes epoll's one-at-a-time model
- **High throughput**: Batched submission + completion, zero-copy
- **Storage I/O**: No contest. io_uring + O_DIRECT + IOPOLL is the fastest path
- **Mixed workloads**: File I/O + networking in the same ring. epoll can't do file I/O.

## The Real Question

It's not "is io_uring faster?" — it obviously is. The question is whether your workload is I/O-bound enough for the difference to matter. If you're spending most of your CPU on application logic, switching to io_uring won't help much.

If syscall overhead is >10% of your CPU profile, io_uring is a clear win.
