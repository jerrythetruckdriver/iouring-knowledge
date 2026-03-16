# io_uring vs. DPDK for Networking

Different tools. Different layers. Comparing them requires understanding what each actually does.

## What They Are

**io_uring**: Kernel-mediated async I/O. Operations go through the kernel network stack. You get TCP/IP, routing, firewall rules, socket API semantics. The kernel manages the NIC.

**DPDK**: Kernel bypass. Userspace drives the NIC directly via UIO/VFIO. No kernel network stack. You implement your own protocol handling — or use a library that does.

## Architecture Comparison

```
io_uring:                          DPDK:
┌──────────┐                       ┌──────────┐
│ App      │                       │ App      │
│ ┌──────┐ │                       │ ┌──────┐ │
│ │SQ/CQ │ │ mmap                  │ │PMD   │ │ userspace driver
│ └──┬───┘ │                       │ └──┬───┘ │
└────┼─────┘                       └────┼─────┘
     │ io_uring_enter()                 │ MMIO / DMA
┌────┼─────┐                       ┌────┼─────┐
│ Kernel   │                       │ NIC HW   │
│ TCP/IP   │                       │          │
│ driver   │                       │          │
└──────────┘                       └──────────┘
```

## Performance Characteristics

| Metric | io_uring | DPDK |
|--------|----------|------|
| Latency floor | ~1-2μs (syscall + kernel stack) | ~100-500ns (poll mode) |
| Packets/sec (single core) | ~2-5M | ~15-40M |
| CPU efficiency | Sleeps when idle | 100% poll (one core dedicated) |
| Zero-copy TX | Yes (6.0+ `IORING_OP_SEND_ZC`) | Yes (native) |
| Zero-copy RX | Yes (6.15+ zcrx) | Yes (native) |
| TCP support | Full kernel TCP | Roll your own or use lib (TLDK, mTCP) |
| UDP support | Full | Full (easier — no state) |

## When io_uring Wins

1. **You need TCP**: Implementing TCP in userspace is hard and usually worse than the kernel's battle-tested stack. TLS, congestion control, socket API compatibility — all free with io_uring.

2. **Mixed workloads**: io_uring handles file I/O, timers, signals, and networking in one ring. DPDK only does networking — you need a separate I/O path for everything else.

3. **CPU cost matters**: DPDK's poll mode driver burns 100% of a core whether packets arrive or not. io_uring sleeps when idle. On a server doing 10k req/s, DPDK wastes a core. io_uring uses fractional CPU.

4. **Container/cloud deployment**: DPDK needs hugepages, VFIO/UIO, dedicated NICs, specific IOMMU config. io_uring works with standard sockets and any NIC.

5. **Maintainability**: Socket API is well-understood. DPDK requires specialized knowledge.

## When DPDK Wins

1. **Raw packet throughput**: If you're building a switch, load balancer, or firewall that moves millions of packets per second per core, DPDK is the tool.

2. **Latency-critical paths**: Sub-microsecond latency requirements. Financial trading, real-time telemetry.

3. **Custom protocols**: If you're not using TCP (e.g., QUIC with custom congestion control, custom UDP protocols), bypassing the kernel stack eliminates overhead you'd never use.

4. **Dedicated hardware**: Purpose-built appliances with NICs assigned to DPDK. No sharing concerns.

## The Hybrid Approach

Some systems use both:

- **DPDK for fast path**: Packet classification, forwarding, simple stateless operations
- **io_uring for slow path**: TCP connections to backends, logging, configuration, metrics

This is what projects like Envoy and HAProxy explore — DPDK for the edge, kernel stack for complexity.

## XDP: The Middle Ground

XDP (eXpress Data Path) sits between io_uring and DPDK:

- Runs BPF programs at the NIC driver level
- No kernel bypass — works within the kernel
- Can redirect packets to userspace via AF_XDP sockets
- AF_XDP + io_uring is possible but immature

```
DPDK (fastest, most complex, kernel bypass)
  │
  ▼
XDP/AF_XDP (fast, BPF-based, kernel-integrated)
  │
  ▼
io_uring (good performance, full kernel stack, easiest)
  │
  ▼
epoll (baseline, proven, everywhere)
```

## io_uring's Trajectory

With each kernel release, io_uring chips away at DPDK's advantages:

- **6.0**: Zero-copy send
- **6.12**: NAPI busy-poll integration
- **6.15**: Zero-copy receive (zcrx)
- **Future**: Ring-mapped RX buffers, more NIC offloads

io_uring won't match DPDK's raw PPS numbers. But for most server applications, it doesn't need to. The kernel network stack is an asset, not a liability.

## Bottom Line

Use DPDK if you're building a network appliance that processes millions of packets per core.

Use io_uring for everything else. The performance gap narrows every kernel release, and you keep TCP, TLS, routing, and your sanity.
