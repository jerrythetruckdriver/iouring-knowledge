# NAPI Busy Poll

io_uring has first-class NAPI busy polling support since kernel 6.9 (`IORING_REGISTER_NAPI`). This is the integration point between io_uring's completion model and the kernel networking stack's packet processing.

## What NAPI Busy Polling Does

Normal path: NIC fires interrupt → softirq runs NAPI poll → packets delivered to socket → io_uring CQE posted.

Busy poll path: io_uring wait call directly invokes NAPI poll on the NIC's receive queue. No interrupt, no softirq. Packets go from NIC to userspace with minimal kernel involvement.

Latency delta: interrupt-driven path adds 5-50µs of scheduling noise. Busy poll eliminates it at the cost of CPU.

## Registration

```c
struct io_uring_napi napi = {
    .busy_poll_to = 100,        /* microseconds to busy poll */
    .prefer_busy_poll = 1,      /* pledge to poll regularly, keep IRQs masked */
};

io_uring_register(ring_fd, IORING_REGISTER_NAPI, &napi, 0);
```

### Fields

| Field | Description |
|-------|-------------|
| `busy_poll_to` | Busy poll timeout in microseconds. How long each wait will spin checking for packets. |
| `prefer_busy_poll` | If set, pledge to the kernel that you'll poll regularly. Kernel keeps device IRQs masked permanently. Revoked if `gro_flush_timeout` passes without a poll. |

### Unregister

```c
struct io_uring_napi napi = {};
io_uring_register(ring_fd, IORING_UNREGISTER_NAPI, &napi, 0);
/* napi.busy_poll_to contains the previous timeout on return */
```

## How It Works

1. Application calls `io_uring_enter()` with `IORING_ENTER_GETEVENTS`
2. Instead of sleeping on a waitqueue, kernel checks if any pending io_uring operations are associated with NAPI IDs
3. Calls `napi_busy_loop()` on each associated NAPI instance
4. NAPI poll fetches packets directly from the NIC's DMA ring
5. Packets flow through the network stack inline
6. Completion events posted to CQ

Key: io_uring tracks which NAPI IDs are associated with its registered sockets. When you do multishot recv on a socket, io_uring knows which NIC queue feeds it.

## NAPI ID Tracking

Every socket has a NAPI ID (queryable via `SO_INCOMING_NAPI_ID`). io_uring maintains a list of NAPI IDs it should poll. This happens automatically when you submit network operations.

For best results, all sockets on a ring should map to the same NAPI ID (same NIC queue). If sockets span multiple queues, io_uring polls all of them, which dilutes the benefit.

## When To Use

**Good fit:**
- Sub-10µs latency requirements (HFT, real-time)
- Dedicated network processing cores (can afford 100% CPU)
- Known NIC queue → core mapping (RSS/RFS configured)
- `SQPOLL` + `IOPOLL` + NAPI = triple zero: zero syscall, zero interrupt, zero context switch

**Bad fit:**
- General purpose servers (wastes CPU on idle connections)
- Mixed workloads on same core
- Containers with shared NICs (no NIC queue control)

## Comparison with Socket-Level Busy Poll

| Mechanism | Scope | Kernel | Registration |
|-----------|-------|--------|-------------|
| `SO_BUSY_POLL` | Per-socket | 3.11+ | `setsockopt()` |
| `net.core.busy_poll` | System-wide | 3.11+ | sysctl |
| `epoll` EPIOCSPARAMS | Per-epoll context | 6.x | ioctl |
| `IORING_REGISTER_NAPI` | Per-ring | 6.9 | `io_uring_register()` |

io_uring's approach is cleaner: one registration covers all sockets on the ring, and it integrates with the completion model rather than bolting onto poll/select/epoll.

## Performance Configuration

```
# RSS: pin NIC queues to specific cores
ethtool -X eth0 equal 4

# IRQ affinity: match NIC queue IRQs to io_uring cores  
echo 4 > /proc/irq/<irq>/smp_affinity_list

# Disable generic IRQ coalescing (we're busy polling)
ethtool -C eth0 rx-usecs 0
```

Combine with:
- `IORING_SETUP_SQPOLL` — no submission syscalls
- `IORING_SETUP_COOP_TASKRUN` — no IPI reschedules
- `IORING_SETUP_SINGLE_ISSUER` — single-thread optimizations
- Registered buffers + provided buffer rings — zero-copy recv path

## Integration with zcrx

NAPI busy poll + zero-copy receive (6.15+) is the endgame for network I/O:
- NAPI busy poll: packets processed without interrupts
- zcrx: packet data stays in NIC-mapped memory, no copy to userspace buffers
- Combined: NIC DMA → mapped pages → io_uring CQE, with zero copies and zero interrupts

This is the path that closes the gap with DPDK for many workloads.
