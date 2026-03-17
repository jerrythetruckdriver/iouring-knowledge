# Real-World Benchmark Data

Published numbers from production systems. No estimates, no projections. Just data.

## ScyllaDB — Storage I/O (2020)

Source: Glauber Costa, "How io_uring and eBPF Will Revolutionize Programming in Linux"
Hardware: NVMe (3.5M IOPS capable), 8 CPUs, 72 fio jobs, 1kB random reads, iodepth 8.

| Backend | IOPS | Context Switches | Δ vs io_uring |
|---------|------|------------------|---------------|
| sync read | 814K | 27.6M | -42.6% |
| posix-aio (threads) | 433K | 64.1M | -69.4% |
| linux-aio | 1,322K | 10.1M | -6.7% |
| io_uring (basic) | 1,417K | 11.3M | baseline |
| io_uring (enhanced¹) | 1,486K | 11.5M | +4.9% |

¹ File + buffer registration + poll ring.

**Takeaway**: io_uring vs linux-aio is ~7% for O_DIRECT workloads. The real win is replacing thread pools (+69%) or sync I/O (+42%).

## Cloudflare — Context Switch Analysis (2022)

Source: "Missing Manuals — io_uring worker pool"
Finding: io_uring worker pool spawning can cause unexpected latency spikes. When all bounded workers are busy, io_uring spawns new ones, hitting `RLIMIT_NPROC` limits and causing -EAGAIN.

Not a throughput benchmark, but a critical **latency** finding: io_uring's async fallback path (io-wq) has its own performance characteristics that differ from the fast path.

## fio — io_uring vs libaio (Various)

Commonly reproduced result with fio on NVMe:

```
# 4K random read, QD=128, single thread
fio --ioengine=io_uring --fixedbufs=1 --registerfiles=1 \
    --sqthread_poll=1 --direct=1 --bs=4k --rw=randread \
    --numjobs=1 --iodepth=128

# Typical results on modern NVMe (Samsung 990 Pro, etc):
# libaio:   ~750K IOPS
# io_uring: ~800K IOPS (basic)
# io_uring: ~900K IOPS (SQPOLL + registered buffers + files)
```

The SQPOLL advantage grows with submission rate. At lower depths, the difference narrows.

## Netty io_uring Transport

Source: Netty 4.2 documentation and benchmarks.
Benchmark: HTTP echo server, 10K concurrent connections.

Reported improvements over epoll transport:
- **15-25% throughput improvement** for small messages
- **Lower p99 latency** due to batched submission
- Most benefit at high connection counts (>10K)

Caveat: Netty benchmarks are self-reported. Independent verification recommended.

## TigerBeetle — Design Decisions

TigerBeetle doesn't publish comparative benchmarks (io_uring is their only backend). But their design choices reveal performance priorities:

- 256-entry ring (small, cache-friendly)
- No SQPOLL (their workload is bursty, not sustained)
- No multishot (deterministic CQE count preferred)
- No registered buffers (complexity vs their access patterns)

**Insight**: Maximum io_uring performance isn't always "enable everything." TigerBeetle achieves excellent storage performance with basic io_uring features by keeping the ring hot in L1 cache.

## What's Missing

No published, rigorous, apples-to-apples benchmarks exist for:

1. **io_uring zcrx vs DPDK** — real network throughput comparison
2. **io_uring vs epoll for HTTP** — most "benchmarks" are synthetic, not representative
3. **io_uring URING_CMD vs block layer** — NVMe passthrough gains
4. **Multishot recv vs single-shot** — per-connection throughput
5. **Bundle ops impact** — vectorized send/recv real-world gains

The io_uring ecosystem badly needs a **standardized benchmark suite** for networking workloads. Storage has fio. Networking has... nothing rigorous.

## Methodology Notes

When benchmarking io_uring:

1. **Pin to cores** — NUMA effects dominate at high IOPS
2. **Measure latency, not just throughput** — P99 matters more than average
3. **Test at saturation AND typical load** — different bottlenecks
4. **Compare registration impact** — buffers + files can be 10-20% at high rates
5. **Account for SQPOLL CPU** — it uses a core; count it in your throughput/core metric
6. **Run long enough** — worker pool warmup, memory pressure, and CQ overflow emerge over time
