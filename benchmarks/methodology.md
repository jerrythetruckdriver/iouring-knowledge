# Benchmarking io_uring: Methodology Guide

## The Problem With Most io_uring Benchmarks

90% of "io_uring vs epoll" benchmarks are flawed. Common mistakes:

1. Comparing single-threaded io_uring against multi-threaded epoll
2. Not warming up registered resources
3. Measuring setup cost instead of steady-state
4. Using tiny payloads where syscall overhead dominates
5. Not pinning threads to cores
6. Ignoring tail latency

## fio: The Standard Tool

fio with `--ioengine=io_uring` is the baseline for storage benchmarks.

### Basic Configuration

```ini
[global]
ioengine=io_uring
direct=1
bs=4k
iodepth=128
numjobs=4
cpus_allowed_policy=split
time_based
runtime=60
ramp_time=5

[randread]
rw=randread
filename=/dev/nvme0n1
```

### io_uring-Specific fio Options

```ini
# Register files (skip fget/fput per I/O)
registerfiles=1

# Register buffers (skip get_user_pages per I/O)
fixedbufs=1

# Use SQPOLL
sqthread_poll=1

# Use IOPOLL (requires O_DIRECT + NVMe)
hipri=1

# Submission batching
iodepth_batch_submit=32
iodepth_batch_complete_min=1
```

### Comparison Matrix

Run all combinations:

| ioengine | Config | What It Tests |
|---|---|---|
| `io_uring` | default | Baseline io_uring |
| `io_uring` | registerfiles + fixedbufs | Registered resources |
| `io_uring` | sqthread_poll | SQPOLL mode |
| `io_uring` | hipri | IOPOLL mode |
| `io_uring` | all of above | Maximum optimization |
| `libaio` | default | Linux AIO baseline |
| `posixaio` | default | POSIX AIO baseline |
| `sync` | default | Synchronous I/O baseline |

## perf-stat for Syscall Counting

```bash
# Count syscalls during benchmark
perf stat -e 'syscalls:sys_enter_io_uring_enter' \
          -e 'syscalls:sys_enter_epoll_wait' \
          -e 'syscalls:sys_enter_read' \
          -e 'syscalls:sys_enter_write' \
          -p <pid> -- sleep 10
```

This tells you the real syscall reduction. io_uring's win is measurable here.

## Tracepoint-Based Latency

```bash
# Per-operation latency histogram
bpftrace -e '
tracepoint:io_uring:io_uring_submit_req { @start[args->req] = nsecs; }
tracepoint:io_uring:io_uring_complete {
    if (@start[args->req]) {
        @latency_us = hist((nsecs - @start[args->req]) / 1000);
        delete(@start[args->req]);
    }
}'
```

## Network Benchmarks

### The Right Way

```bash
# Server: use a framework that supports both epoll and io_uring
# Client: wrk2 with constant request rate (not max throughput)
wrk2 -t4 -c100 -d60s -R50000 --latency http://server:8080/

# Measure:
# 1. Throughput at fixed concurrency
# 2. Latency percentiles at fixed request rate
# 3. CPU usage at both
```

### What to Report

| Metric | Why |
|---|---|
| Requests/sec at fixed concurrency | Raw throughput |
| P50/P99/P999 at fixed rate | Tail latency |
| CPU utilization at fixed rate | Efficiency |
| Syscalls/sec | io_uring's actual reduction |
| Context switches/sec | Thread scheduling overhead |

## Common Pitfalls

### 1. NUMA Effects

```bash
# Pin to a single NUMA node
numactl --cpunodebind=0 --membind=0 ./benchmark
```

Cross-NUMA memory access adds 50-100ns per access. If your io_uring buffers are on node 1 but the thread runs on node 0, you're benchmarking NUMA, not io_uring.

### 2. CPU Frequency Scaling

```bash
# Lock CPU frequency
for cpu in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do
    echo performance > $cpu
done
```

### 3. IRQ Affinity

```bash
# Steer NIC interrupts to the benchmark cores
echo <cpu_mask> > /proc/irq/<irq>/smp_affinity
```

### 4. Kernel Version

Always report kernel version. io_uring behavior changes significantly between releases:
- 5.x: Basic ops, no COOP_TASKRUN
- 6.0-6.6: SINGLE_ISSUER, improved SQPOLL
- 6.7+: NO_SQARRAY, futex, mixed CQEs
- 6.13+: Hybrid IOPOLL, ring resize, registered wait

### 5. Warm-Up

Register resources, submit 1000+ requests, drain them, **then** start timing. First-touch page faults and JIT effects distort results.

## Reporting Template

```
## Environment
- Kernel: 6.19.0
- CPU: AMD EPYC 9654 (96 cores)
- NVMe: Samsung PM9A3 3.84TB
- Memory: 256GB DDR5-4800
- fio: 3.36
- liburing: 2.14

## Configuration
- [paste fio job file]

## Results
| Metric | io_uring | io_uring (optimized) | libaio | sync |
|--------|----------|---------------------|--------|------|
| IOPS   |          |                     |        |      |
| BW     |          |                     |        |      |
| P99 μs |          |                     |        |      |
| CPU %  |          |                     |        |      |
| syscalls/s |      |                     |        |      |
```

## Tools Summary

| Tool | Purpose |
|---|---|
| `fio` | Storage I/O benchmarks |
| `wrk2` | HTTP latency benchmarks (constant rate) |
| `perf stat` | Syscall counting, CPU counters |
| `bpftrace` | Per-request latency histograms |
| `numactl` | NUMA placement |
| `turbostat` | CPU frequency verification |
| `ethtool -S` | NIC stats (for NAPI benchmarks) |
