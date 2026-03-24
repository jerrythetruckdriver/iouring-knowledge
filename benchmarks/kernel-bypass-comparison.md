# Kernel Bypass Comparison: io_uring vs DPDK vs XDP vs AF_XDP

## Architecture Overview

| Approach | Layer | Kernel Involvement | Memory Model |
|----------|-------|-------------------|-------------|
| io_uring | Syscall batching | Full kernel stack | Shared rings (mmap) |
| DPDK | Full bypass | None (userspace NIC driver) | Hugepage pools |
| XDP | In-kernel fast path | BPF program at driver level | Per-CPU pages |
| AF_XDP | Partial bypass | XDP redirect to userspace | UMEM shared rings |

## Detailed Comparison

### io_uring

**How it works**: Batches syscalls via shared memory rings. Kernel still processes packets through full network stack (TCP/IP, conntrack, iptables, socket buffers).

**Strengths**:
- Works with any socket — TCP, UDP, Unix, SCTP
- Full kernel networking stack (routing, firewall, QoS)
- No special NIC drivers needed
- Incremental adoption (add to existing app)
- Security model intact (user/kernel separation)

**Weaknesses**:
- Can't beat kernel stack overhead for raw throughput
- Per-packet: still ~1µs through the stack
- Not competitive at 100Gbps line rate

**Best for**: Application-level networking (HTTP, gRPC, databases, proxies).

### DPDK

**How it works**: Userspace NIC driver (PMD). Application directly reads/writes NIC DMA rings. No kernel involvement for data path.

**Strengths**:
- Highest raw throughput (100Gbps+ line rate)
- Lowest latency (~100ns per packet)
- Predictable performance (no kernel jitter)

**Weaknesses**:
- Dedicated NIC(s) — can't share with kernel stack
- No TCP (must implement in userspace, or use F-Stack/mTCP)
- Hugepage memory requirements (often 1GB+)
- Root/capabilities required
- Complex deployment (VFIO/UIO setup, NUMA pinning)
- No firewall, routing, or any kernel networking feature

**Best for**: Network functions (firewalls, load balancers, routers), telco, financial trading.

### XDP (eXpress Data Path)

**How it works**: BPF program runs at NIC driver level, before `sk_buff` allocation. Can drop, redirect, modify, or pass packets up. Kernel-managed.

**Strengths**:
- Near-DPDK speed for simple operations (drop, redirect)
- Shares NIC with kernel stack (pass-through for non-matched traffic)
- No special drivers (native mode in modern NICs, generic mode everywhere)
- eBPF safety guarantees (verifier, no crashes)

**Weaknesses**:
- Limited to per-packet processing (no connection state in XDP itself)
- BPF program constraints (no loops pre-6.x, limited stack, verifier limits)
- No userspace access to packet data (that's AF_XDP's job)
- Can't do TCP/UDP reassembly

**Best for**: DDoS mitigation, packet filtering, load balancer steering, traffic sampling.

### AF_XDP

**How it works**: XDP redirects selected packets to a userspace socket (AF_XDP). Shared UMEM ring between kernel and user. Zero-copy in native mode.

**Strengths**:
- DPDK-like performance with kernel cooperation
- Shares NIC with kernel (XDP selects which packets to redirect)
- Zero-copy in supported NICs
- Simpler than DPDK (no custom drivers)

**Weaknesses**:
- Raw packets only (no TCP — must handle IP/UDP yourself)
- UMEM management complexity
- Not all NICs support zero-copy mode
- Still limited ecosystem vs DPDK

**Best for**: High-throughput UDP processing, DNS servers, custom protocol engines, telemetry collection.

## Performance Positioning

```
Throughput (packets/sec, single core):

DPDK:       ~40-60 Mpps  (raw L2 forwarding)
AF_XDP:     ~20-30 Mpps  (zero-copy mode)
XDP:        ~20-40 Mpps  (in-kernel processing)
io_uring:   ~2-5 Mpps    (full kernel TCP stack)
epoll:      ~1-3 Mpps    (full kernel TCP stack)

Note: io_uring vs epoll gap narrows with connection count.
      io_uring wins on syscall reduction, not per-packet speed.
```

## io_uring's Trajectory

io_uring is closing the gap in specific areas:

1. **Zero-copy receive (zcrx, 6.15)**: NIC DMA → userspace buffer, no copy. Approaching AF_XDP for received data.
2. **NAPI busy polling (6.9)**: Skip interrupt path, poll NIC directly. Reduces latency toward DPDK.
3. **DMA-BUF zcrx (6.16)**: GPU/NPU direct networking without CPU involvement.
4. **Bundle ops**: Batch multiple recv/send into one SQE. Amortizes per-op overhead.

The endgame: io_uring + zcrx + NAPI + SQPOLL + registered buffers approaches AF_XDP performance while keeping TCP/UDP kernel stack support.

## Decision Matrix

| Requirement | io_uring | DPDK | XDP | AF_XDP |
|------------|----------|------|-----|--------|
| TCP support | ✅ Native | ❌ DIY | ❌ | ❌ |
| UDP support | ✅ Native | ❌ DIY | ✅ Process | ✅ Raw |
| > 10 Mpps | ❌ | ✅ | ✅ | ✅ |
| Share NIC with kernel | ✅ | ❌ | ✅ | ✅ |
| No special setup | ✅ | ❌ | ~✅ | ~✅ |
| Application server | ✅ | ❌ Wrong tool | ❌ | ❌ |
| Network function | ❌ Too slow | ✅ | ✅ | ✅ |
| Incremental migration | ✅ | ❌ Rewrite | ~✅ | ❌ |

## Hybrid Architectures

Real systems often combine approaches:

- **CDN edge**: XDP for DDoS → kernel TCP → io_uring for application I/O
- **Trading**: DPDK for market data → io_uring for order management
- **DNS**: AF_XDP for query processing → io_uring for upstream TCP resolution
- **Load balancer**: XDP for L4 steering → io_uring for L7 processing

io_uring's role is almost always the application layer. Kernel bypass handles the wire.
