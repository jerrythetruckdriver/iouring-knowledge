# Ring Topology — Per-Thread vs Per-Connection vs Shared

## The Three Models

### 1. Ring-Per-Thread (Recommended)

One ring per thread/core. All connections handled by that thread share one ring.

```
Thread 0: ring_0 → 10,000 connections
Thread 1: ring_1 → 10,000 connections
Thread N: ring_N → 10,000 connections
```

**Setup flags:** `SINGLE_ISSUER | COOP_TASKRUN | NO_SQARRAY`

**Pros:**
- Optimal kernel fast-path (SINGLE_ISSUER enables lockless submission)
- COOP_TASKRUN avoids task_work IPIs
- Registered resources amortized across all connections
- Memory overhead: O(threads), not O(connections)

**Cons:**
- Connection distribution matters (RSS or MSG_RING dispatch)
- Hot thread = hot ring (uneven load)

**Who uses it:** ScyllaDB, Redpanda, TigerBeetle, Glommio, Monoio, Seastar

### 2. Ring-Per-Connection

One ring per connection. Maximum isolation.

```
conn_0: ring_0
conn_1: ring_1
conn_N: ring_N
```

**Pros:**
- Perfect isolation (one slow connection can't stall others)
- Simple cancellation (destroy ring = cancel everything)
- No contention

**Cons:**
- Memory: ~6-20KB per ring × N connections. 10K connections = 60-200MB just for rings
- Kernel overhead: N rings means N io_uring instances to manage
- Can't use SINGLE_ISSUER if connections migrate between threads
- Registered resources duplicated per ring (or use CLONE_BUFFERS)
- Poor batching: 1 SQE per submission instead of N

**When it makes sense:** Very few long-lived connections with heavy I/O each (database connections to storage). **Not** for high-connection-count servers.

### 3. Shared Ring (Multiple Threads)

One ring, multiple submitter threads.

```
Thread 0 ─┐
Thread 1 ─┼─→ shared_ring → all connections
Thread N ─┘
```

**Setup flags:** Cannot use `SINGLE_ISSUER`. Need `SQPOLL` or explicit locking.

**Pros:**
- Simple architecture
- Single point for resource registration

**Cons:**
- No SINGLE_ISSUER = slower submission path (atomic operations)
- No COOP_TASKRUN = task_work IPIs
- Contention on SQ tail under load
- Poor NUMA locality

**Who uses it:** Ceph BlueStore (mutex-protected), legacy codebases transitioning from epoll

## Decision Matrix

| Factor | Per-Thread | Per-Connection | Shared |
|--------|-----------|----------------|--------|
| Connections | 1K-1M | <100 | Any |
| Memory | Low | High | Lowest |
| Batching | Excellent | Poor | Good |
| Isolation | Per-thread | Per-connection | None |
| SINGLE_ISSUER | ✅ | ✅ (if pinned) | ❌ |
| Complexity | Medium | Low | Low |
| Performance | **Best** | Worst | Middle |

## Hybrid: Ring-Per-Thread + MSG_RING

The production pattern combines ring-per-thread with MSG_RING for cross-thread work:

1. Accept thread receives connection via multishot accept
2. Sends fd to target thread's ring via `IORING_OP_MSG_RING` with `IORING_MSG_RING_FD`
3. Target thread owns the connection for its lifetime
4. REGISTER_SEND_MSG_RING for signaling from non-ring threads (worker pools)

This gives you per-thread isolation with cross-thread coordination. It's what thread-per-core architectures converge on.

## Ring Sizing by Topology

| Topology | Typical SQ Size | Typical CQ Size |
|----------|----------------|-----------------|
| Per-thread (10K conns) | 4096 | 8192-16384 |
| Per-connection | 32-64 | 64-128 |
| Shared (N threads) | 4096-8192 | 16384-32768 |

Use `IORING_REGISTER_RESIZE_RINGS` (6.13) to start small and grow.

## Anti-Patterns

- **Ring-per-request**: Creating/destroying rings for individual operations. Ring setup is not free (~microseconds). Use one ring, submit multiple SQEs.
- **Shared ring without SQPOLL**: If you're sharing a ring across threads without SQPOLL, you're just adding contention for no benefit. Use per-thread rings.
- **Per-connection with registered buffers**: Pinning 64KB of buffers × 10K connections = 640MB of locked memory. Use provided buffer rings shared across the thread's ring instead.
