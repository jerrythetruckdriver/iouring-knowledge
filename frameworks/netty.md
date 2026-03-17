# Netty io_uring Transport

## Status

Graduated from incubator to mainline **Netty 4.2**. The `netty-incubator-transport-io_uring` repo is archived. io_uring is now a first-class transport alongside epoll and kqueue.

**Artifact:** `io.netty:netty-transport-native-io_uring`
**Platforms:** x86_64, aarch64 (riscv64 in progress)
**Minimum kernel:** 5.14

## Architecture

Netty's io_uring transport maps the Netty Channel model onto io_uring:

- **One ring per EventLoop thread** — matches Netty's one-thread-per-group model
- **SQE submission** via JNI into liburing
- **CQE harvesting** on the event loop tick
- **Fixed files** for accepted connections (avoids fget/fput per operation)
- **Provided buffers** for recv operations

### Channel Types

| Netty Channel | io_uring Mapping |
|---------------|-----------------|
| `IoUringServerSocketChannel` | ACCEPT (multishot) |
| `IoUringSocketChannel` | RECV / SEND |
| `IoUringDatagramChannel` | RECVMSG / SENDMSG |

### Submission Path

```
Channel.write(buf)
  → IoUringIoHandler.prepSend(sqe, fd, buf)
  → io_uring_submit() at end of event loop tick (batched)
```

All SQEs for an event loop iteration are batched into a single `io_uring_enter()` call. This is io_uring's strength — Netty's epoll transport needs one `epoll_ctl` per registration change plus one `epoll_wait` per tick. io_uring collapses everything into one syscall.

## Performance Characteristics

### Where io_uring Wins

- **High connection accept rates:** Multishot accept means one SQE handles all incoming connections. Epoll needs re-arming.
- **Batched submissions:** 100 sends = 1 syscall. Epoll: 100 writes + 1 epoll_wait = 101 syscalls.
- **Fixed file descriptors:** Skip atomic refcount on every I/O op.

### Where Epoll is Fine

- **Low connection counts:** The per-op overhead savings don't matter at 100 connections.
- **Simple request-response:** One recv, one send, repeat. The batching advantage disappears.
- **Kernel < 5.14:** No io_uring transport available.

### Published Numbers

Netty hasn't published official io_uring vs epoll benchmarks. Community reports suggest:
- 10-20% throughput improvement on echo benchmarks at high concurrency (10K+ connections)
- Lower tail latency under load (fewer syscalls = fewer context switches)
- Memory savings from provided buffer rings vs per-connection allocated buffers

These are workload-dependent. Don't expect miracles on a CRUD app doing 100 req/sec.

## JNI Bridge

Netty uses a thin JNI layer to call liburing. The native code:
- Sets up the ring
- Provides prep helpers for each operation type
- Handles the mmap'd ring memory

The JNI overhead is amortized across batched submissions. One JNI transition per event loop tick, not per operation.

## Gotchas

- **memlock limits:** io_uring needs locked memory for the rings. Default `ulimit -l` (64K) is often too small. Increase to at least 8MB for production.
- **Kernel version:** Features are kernel-dependent. Multishot accept needs 5.19+, provided buffer rings need 5.19+.
- **Docker:** Default seccomp profile blocks io_uring. Need custom profile or `--security-opt seccomp=unconfined`.
- **Not a drop-in replacement:** Need to change channel types in bootstrap. Same Netty API otherwise.
