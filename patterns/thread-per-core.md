# Thread-per-Core Architecture with io_uring

## The Model

One thread per CPU core. One io_uring ring per thread. No shared state. No locks. No cross-thread synchronization in the hot path.

This is the architecture used by ScyllaDB, Redpanda, TigerBeetle, Glommio, and Monoio. It's where io_uring shines hardest.

## Why io_uring Fits Thread-per-Core

**epoll** works fine with thread-per-core, but io_uring adds:

1. **SINGLE_ISSUER + COOP_TASKRUN**: Kernel knows only one thread touches the ring. Eliminates atomic ops and IPI overhead.
2. **Registered resources per ring**: Each thread has its own registered files and buffers. No sharing, no coordination.
3. **SQPOLL per thread**: Dedicated kernel polling thread per core, pinned via `sq_thread_cpu`.
4. **No syscalls in hot path**: With SQPOLL + IOPOLL + registered everything, the data path is purely userspace.

## Ring Configuration

```c
struct io_uring_params p = {
    .flags = IORING_SETUP_SQPOLL
           | IORING_SETUP_COOP_TASKRUN
           | IORING_SETUP_SINGLE_ISSUER
           | IORING_SETUP_NO_SQARRAY,
    .sq_thread_cpu = core_id,  // pin SQPOLL to same core
    .sq_thread_idle = 1000,    // 1ms idle before sleeping
};
```

For storage-heavy workloads, add `IORING_SETUP_IOPOLL`.

## Per-Thread Resource Registration

Each thread registers its own:

```c
// Fixed files: pre-opened fds for this shard
io_uring_register_files(&ring, shard_fds, nr_fds);

// Fixed buffers: pinned memory for this shard's I/O
io_uring_register_buffers(&ring, shard_bufs, nr_bufs);

// Provided buffer ring: for recv/accept
io_uring_register_buf_ring(&ring, &buf_reg, 0);

// NAPI: direct NIC queue polling
io_uring_register_napi(&ring, &napi);
```

## Connection Distribution

### Accept on one ring, dispatch via MSG_RING

```
Acceptor thread (core 0):
  multishot ACCEPT → new fd
  MSG_RING → send fd to target core's ring (round-robin or hash)

Worker thread (core N):
  receives fd via CQE from MSG_RING
  FIXED_FD_INSTALL → registers in own file table
  multishot RECV on the connection
```

### RSS/RPS Hardware Distribution

Better: configure NIC RSS (Receive Side Scaling) to steer connections to specific RX queues, one per core. Each core's ring does its own accept on a shared listen socket with `SO_REUSEPORT`.

```
Core 0: listen + accept → connections hashed to core 0 by NIC
Core 1: listen + accept → connections hashed to core 1 by NIC
...
```

No cross-core communication for connection setup.

## Memory Architecture

### Per-Core Allocation

```
Core 0: [ring SQ/CQ] [registered bufs] [provided buf ring] [app data]
Core 1: [ring SQ/CQ] [registered bufs] [provided buf ring] [app data]
...
```

All allocations NUMA-local:
```c
void *buf = numa_alloc_onnode(size, numa_node_of_cpu(core_id));
```

### Clone Buffers for Shared Read-Only Data

If multiple cores need the same registered buffers (e.g., static file cache):
```c
// Register on core 0, clone to core 1
io_uring_register_clone_buffers(&ring_core1, &clone_args);
```

Pages are shared (COW), metadata is per-ring. Saves memory without sharing mutable state.

## Inter-Thread Communication

### MSG_RING (fast, CQE-based)

```c
// Thread A → Thread B: post a data CQE
io_uring_prep_msg_ring(sqe, ring_b_fd, result, user_data, 0);
```

Latency: ~100-200ns. No syscall if SQPOLL is active. Recipient sees it as a normal CQE.

### REGISTER_SEND_MSG_RING (from non-ring threads)

For threads that don't have their own ring (e.g., timer threads, control plane):
```c
io_uring_register_send_msg_ring(ring_fd, &msg);
```

### Futex (for synchronization primitives)

```c
// Thread A: wake thread B's futex
io_uring_prep_futex_wake(sqe, &shared_futex, 1, FUTEX2_SIZE_U32, 0);
```

Async mutex/condvar without blocking the event loop.

## Event Loop Pattern

```c
void shard_main(int core_id) {
    cpu_set_t cpuset;
    CPU_SET(core_id, &cpuset);
    pthread_setaffinity_np(pthread_self(), sizeof(cpuset), &cpuset);

    struct io_uring ring;
    io_uring_queue_init_params(SQ_SIZE, &ring, &params);

    // Register everything
    register_resources(&ring, core_id);

    // Event loop — no syscalls with SQPOLL
    while (running) {
        // Prep new SQEs based on application state
        prep_submissions(&ring);

        // Submit (SQPOLL: just advance tail, no syscall)
        io_uring_submit(&ring);

        // Reap completions
        struct io_uring_cqe *cqe;
        unsigned head;
        io_uring_for_each_cqe(&ring, head, cqe) {
            handle_completion(cqe);
        }
        io_uring_cq_advance(&ring, nr_cqes);

        // If nothing ready, wait with registered wait
        if (nr_cqes == 0) {
            io_uring_submit_and_wait_reg(&ring, &wait_arg, 1);
        }
    }
}
```

## Who Does This

| Project | Language | Threads | Rings | Notes |
|---|---|---|---|---|
| ScyllaDB | C++ (Seastar) | Per-core | Per-shard | 1M+ IOPS per node |
| Redpanda | C++ (Seastar) | Per-core | Per-shard | Kafka-compatible streaming |
| TigerBeetle | Zig | Per-core | Per-thread | Financial transactions |
| Glommio | Rust | Per-core | Per-executor | Runtime library |
| Monoio | Rust | Per-core | Per-runtime | ByteDance production |

## The Trade-Off

**Wins**: Zero contention, predictable latency, maximum throughput per core, cache-friendly.

**Costs**: Data partitioning complexity, cross-shard queries need coordination, uneven load distribution, more memory (per-core buffers), harder to debug than shared-memory.

The model works when your workload is naturally partitionable (connections, keys, shards). It's a poor fit for workloads that need global state (e.g., a single sorted index).
