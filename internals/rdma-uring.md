# io_uring and RDMA

## Status: No URING_CMD for RDMA Verbs (as of 6.19)

No kernel support exists for submitting RDMA verbs via io_uring URING_CMD. The RDMA subsystem (libibverbs, rdma-core) has its own completion model — Completion Queues (CQs) — that predates io_uring by over a decade.

## Why No Integration

**Architectural mismatch:**
- RDMA already has kernel-bypass. That's the whole point. Adding io_uring as an intermediary re-introduces the kernel.
- RDMA CQs are polled in userspace via `ibv_poll_cq()`. No syscall. io_uring can't beat zero syscalls.
- RDMA verbs (post_send, post_recv) write directly to NIC hardware queues via doorbell registers. No kernel involvement.

**Where they overlap:**
- Both io_uring and RDMA solve the same fundamental problem: batched, async I/O without per-operation syscalls.
- Both use submission/completion queue models with shared memory between kernel and userspace.
- io_uring is the "RDMA for everything else" — storage, networking, filesystem ops.

## Bridging io_uring and RDMA

For applications that need both RDMA and regular I/O:

```
Approach 1: POLL_ADD on RDMA completion channel fd
  io_uring_prep_poll_add(sqe, comp_channel_fd, POLLIN);
  // When CQE fires, call ibv_poll_cq() to drain RDMA completions

Approach 2: Separate event loops
  // RDMA thread: busy-poll ibv_poll_cq()
  // io_uring thread: normal io_uring event loop
  // Cross-thread: MSG_RING or futex for signaling
```

POLL_ADD on the RDMA completion channel fd is the practical bridge. You get io_uring handling storage/network I/O and RDMA handling the high-speed fabric path, unified in one event loop.

## RDMA vs io_uring for Networking

| Aspect | RDMA | io_uring |
|--------|------|----------|
| Kernel bypass | Full (verbs go to NIC) | Partial (batched syscalls) |
| Zero-copy | Native (registered MRs) | zcrx (6.15), SEND_ZC |
| Latency | ~1-2μs | ~3-10μs |
| CPU overhead | Near-zero per op | Low but non-zero |
| Hardware | Requires RDMA NIC | Any NIC |
| Portability | InfiniBand, RoCE | Any Linux socket |
| Complexity | High (MR registration, QP state machine) | Medium |
| Ecosystem | HPC, storage fabrics | General purpose |

## The Convergence

io_uring's zcrx + NAPI busy poll + DMA-BUF is closing the gap from the io_uring side. RDMA still wins on raw latency for specialized hardware, but io_uring handles everything else with a single API.

For most applications: use io_uring. Reserve RDMA for InfiniBand fabrics and sub-2μs latency requirements where you already have the hardware.

## See Also

- [NVMe over Fabrics](nvme-of.md) — NVMe-oF can use RDMA transport; io_uring handles the block path transparently
- [Kernel Bypass Comparison](../benchmarks/kernel-bypass-comparison.md) — io_uring vs DPDK vs XDP vs AF_XDP
- [Zero-Copy Receive](../patterns/zero-copy-rx.md) — io_uring zcrx as the non-RDMA zero-copy path
