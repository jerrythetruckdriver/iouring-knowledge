# NUMA Considerations

io_uring on multi-socket systems. Where your memory lives matters.

## The Problem

io_uring's shared rings are memory-mapped regions. On NUMA systems, if the ring memory lands on the wrong node, every SQE write and CQE read crosses the interconnect. That's 100-200ns of latency per access on a 2-socket system.

## Ring Memory Placement

### Default Behavior

`io_uring_setup()` allocates ring memory via the kernel's page allocator. The kernel uses the calling thread's memory policy — usually local allocation. So if you call `io_uring_setup()` from a thread running on node 0, the ring lands on node 0.

**Rule**: Pin your thread to the target CPU/node **before** creating the ring.

```c
cpu_set_t cpus;
CPU_ZERO(&cpus);
CPU_SET(target_cpu, &cpus);
sched_setaffinity(0, sizeof(cpus), &cpus);

// Now create the ring — memory will be node-local
io_uring_queue_init(256, &ring, flags);
```

### Application-Managed Memory (6.5+)

`IORING_SETUP_NO_MMAP` lets you provide pre-allocated memory for the ring:

```c
// Allocate on specific NUMA node
void *sq_ring = mmap(...);
mbind(sq_ring, sq_size, MPOL_BIND, &node_mask, ...);

// Pass to io_uring
params.flags |= IORING_SETUP_NO_MMAP;
// ... set up user_addr fields in sq_off/cq_off
```

Full control over placement. Use with `set_mempolicy()` or `mbind()`.

## io-wq Worker Affinity

io-wq workers are kernel threads that handle operations that can't complete inline. By default, they inherit the ring owner's CPU affinity.

### Controlling Affinity

```c
// Set io-wq worker CPU mask
cpu_set_t mask;
CPU_ZERO(&mask);
// Bind to NUMA node 0 CPUs
for (int i = 0; i < cpus_per_node; i++)
    CPU_SET(node0_cpus[i], &mask);

io_uring_register_iowq_aff(ring, sizeof(mask), &mask);
```

`IORING_REGISTER_IOWQ_AFF` (opcode 17) sets the affinity. `IORING_UNREGISTER_IOWQ_AFF` (opcode 18) resets to default.

**Important**: Worker threads can still be scheduled on any CPU in the mask. They won't be pinned to a single core — the scheduler picks from the allowed set.

### Worker Count Per Node

```c
unsigned int workers[2] = { 4, 4 };  // [bounded, unbounded]
io_uring_register_iowq_max_workers(ring, workers);
```

Worker counts are per-ring, not per-node. On NUMA systems with thread-per-core, each core's ring gets its own worker pool naturally.

## SQPOLL Thread Placement

`IORING_SETUP_SQ_AFF` pins the SQPOLL thread to a specific CPU:

```c
params.flags |= IORING_SETUP_SQPOLL | IORING_SETUP_SQ_AFF;
params.sq_thread_cpu = target_cpu;
```

On NUMA: pick a CPU on the same node as the ring memory and the application thread. Cross-node SQPOLL defeats the purpose.

## Thread-Per-Core on NUMA

The natural NUMA pattern for io_uring:

```
Node 0:                          Node 1:
  CPU 0 → Ring 0 → local mem      CPU 4 → Ring 4 → local mem
  CPU 1 → Ring 1 → local mem      CPU 5 → Ring 5 → local mem
  CPU 2 → Ring 2 → local mem      CPU 6 → Ring 6 → local mem
  CPU 3 → Ring 3 → local mem      CPU 7 → Ring 7 → local mem
```

Each thread:
1. Pinned to one CPU
2. Creates its own ring (memory lands on local node)
3. Sets io-wq affinity to same-node CPUs
4. Registers buffers from local NUMA allocations
5. Uses `SINGLE_ISSUER` — no cross-thread submission

### Buffer Registration

Registered buffers (`IORING_REGISTER_BUFFERS`) should be allocated from the local node:

```c
// Allocate on local node
void *buf = numa_alloc_onnode(size, node);
struct iovec iov = { .iov_base = buf, .iov_len = size };
io_uring_register_buffers(ring, &iov, 1);
```

Buffer clone (`IORING_REGISTER_CLONE_BUFFERS`, 6.13) can share buffer tables between rings, but the underlying pages still live on whatever node allocated them. Clone is for sharing access, not for NUMA placement.

## Shared Work Queue

`IORING_SETUP_ATTACH_WQ` lets multiple rings share a worker pool:

```c
params.flags |= IORING_SETUP_ATTACH_WQ;
params.wq_fd = existing_ring_fd;
```

On NUMA: only share between rings on the **same node**. Cross-node wq sharing means worker threads bounce between nodes. Bad.

## Measurement

Check NUMA stats with:

```bash
# Per-node memory allocation
numastat -p $(pgrep your_app)

# io-wq thread placement
for pid in $(ls /proc/$(pgrep your_app)/task/); do
    echo "Thread $pid: $(taskset -p $pid 2>/dev/null | grep mask)"
done

# Cross-node memory accesses (perf)
perf stat -e node-load-misses,node-store-misses ./your_app
```

`node-load-misses` and `node-store-misses` are your signal. If they're high relative to local accesses, your ring or buffer memory is on the wrong node.

## TL;DR

1. Pin thread before creating ring
2. Use `NO_MMAP` + `mbind()` for explicit placement
3. Set io-wq affinity to same-node CPUs
4. Pin SQPOLL thread to same node
5. Allocate registered buffers from local node
6. Don't share work queues across nodes
7. Measure with `numastat` and `perf stat`
