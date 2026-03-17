# When to Use io_uring

Architecture decision guide. No fluff, just the trade-offs.

## Use io_uring When

### High IOPS Storage
- NVMe with O_DIRECT + IOPOLL
- Batched submissions amortize syscall cost
- Registered buffers eliminate per-I/O DMA mapping
- **Threshold:** >50K IOPS or latency-sensitive storage

### High Connection Count Networking
- Multishot accept + recv eliminates per-event syscalls
- epoll needs 2+ syscalls per event; io_uring batches to ~0
- Bundle ops reduce CQ pressure at scale
- **Threshold:** >10K concurrent connections or >100K req/s

### Linked Operation Chains
- write → fsync → write as one atomic submission
- No scheduling gaps between dependent ops
- Database WAL patterns benefit significantly

### Kernel↔Userspace IPC
- URING_CMD replaces read/write/ioctl on char devices
- ublk, FUSE-over-io_uring prove the pattern
- Multishot URING_CMD for streaming events

### Zero-Copy Networking
- Send: IORING_OP_SEND_ZC (6.0+)
- Receive: zcrx (6.15+), including DMA-BUF for GPU-direct
- Only wins when data is large enough to amortize setup

## Don't Use io_uring When

### Few Concurrent Operations
- Single-threaded CLI tool doing sequential file reads
- io_uring setup cost (~10µs) dominates
- Just use `read()`/`write()`

### Portability Required
- macOS: no io_uring (kqueue only)
- FreeBSD: no io_uring
- Windows: IOCP is the equivalent
- Cross-platform: compio (Rust) abstracts this

### Container/Cloud Environments (Without Checking)
- RHEL 9: io_uring disabled by default
- Docker: seccomp blocks io_uring by default
- Android 15: restricted to first-party apps
- **Always probe before depending on it**

### Simple Event Notification
- If you just need "is this fd readable?" — epoll is simpler
- io_uring's value is batched I/O, not event notification
- IORING_OP_EPOLL_WAIT (6.15) bridges this gap

### CPU-Bound Workloads
- io_uring helps with I/O-bound bottlenecks
- If your bottleneck is computation, io_uring won't help
- SQPOLL wastes a core if I/O is infrequent

## io_uring vs Alternatives

| Scenario | Best Choice | Why |
|----------|------------|-----|
| <1K connections, simple protocol | epoll | Less complexity |
| >10K connections, high throughput | io_uring | Batched submission, zero-copy |
| Packet processing, line rate | DPDK/XDP | Kernel bypass, deterministic latency |
| Mixed I/O + event notification | io_uring | Unified submission queue |
| Cross-platform library | epoll/kqueue + abstraction | io_uring is Linux-only |
| Storage IOPS maximization | io_uring + IOPOLL | Polling eliminates interrupt overhead |
| Userspace block device | ublk (io_uring) | Only viable high-perf option |
| Userspace filesystem | FUSE-over-io_uring | 2-5x faster than /dev/fuse |

## Migration Decision Tree

```
Is your bottleneck I/O?
├─ No → io_uring won't help
└─ Yes
   ├─ Is it storage?
   │  ├─ <50K IOPS → probably not worth it
   │  └─ >50K IOPS → io_uring + registered buffers + IOPOLL
   ├─ Is it networking?
   │  ├─ <1K connections → epoll is fine
   │  ├─ 1K-10K connections → io_uring if you want zero-copy
   │  └─ >10K connections → io_uring multishot
   └─ Is it kernel↔userspace?
      └─ URING_CMD passthrough
```

## The Honest Assessment

io_uring is not universally better. It's better when:
1. You have enough concurrent I/O to amortize ring setup
2. You can use advanced features (registered buffers, multishot, zero-copy)
3. You're on Linux and can require kernel 5.x+
4. Your deployment environment allows it (not RHEL 9 default, not Docker default)

If none of those apply, epoll + traditional syscalls is perfectly fine. Don't over-engineer.
