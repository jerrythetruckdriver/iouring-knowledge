# SO_REUSEPORT + Multishot Accept

## The Problem

`SO_REUSEPORT` creates separate accept queues per socket. Kernel distributes incoming connections via hash (source IP+port) or eBPF program. With epoll, each thread calls `accept()` independently. With io_uring, multishot accept per ring handles this naturally.

## Architecture

```
Thread 0: socket(SO_REUSEPORT) → bind → listen → ring0: multishot ACCEPT
Thread 1: socket(SO_REUSEPORT) → bind → listen → ring1: multishot ACCEPT
Thread 2: socket(SO_REUSEPORT) → bind → listen → ring2: multishot ACCEPT
                    ↑                                         ↑
              Kernel distributes                    Each ring gets its
              by src hash or BPF                    own connection stream
```

## Setup Pattern

Each thread creates its own listening socket with `SO_REUSEPORT`:

```
// Per-thread setup
SOCKET(AF_INET, SOCK_STREAM, 0)
  → SETSOCKOPT(SOL_SOCKET, SO_REUSEPORT, 1) via URING_CMD
  → BIND(addr)
  → LISTEN(backlog)
  → multishot ACCEPT with IORING_FILE_INDEX_ALLOC
```

## Key Details

**Kernel load balancing:**
- Default: hash of (src_ip, src_port, dst_ip, dst_port) → consistent connection affinity
- BPF: `SO_ATTACH_REUSEPORT_EBPF` for custom routing (NUMA-aware, least-loaded, etc.)
- Both work transparently with multishot accept

**Direct descriptors matter here.** Each thread's ring has its own fixed file table. `IORING_FILE_INDEX_ALLOC` auto-allocates into the local table — zero fd table contention across threads.

**ACCEPT_DONTWAIT + ACCEPT_POLL_FIRST:**
- `IORING_ACCEPT_DONTWAIT`: don't block if no connection ready
- `IORING_ACCEPT_POLL_FIRST`: arm poll before attempting accept (reduces failed accept attempts)

## vs MSG_RING Dispatch

Alternative: single listening socket, one thread accepts, MSG_RING dispatches connections.

| Approach | Pros | Cons |
|----------|------|------|
| SO_REUSEPORT + multishot per thread | No cross-thread coordination, kernel-balanced, NUMA-local | N listening sockets, BPF needed for custom balance |
| Single accept + MSG_RING dispatch | One socket, flexible routing | Bottleneck on accept thread, cross-ring MSG_RING overhead |

**Recommendation:** SO_REUSEPORT for most servers. MSG_RING dispatch for connection routing logic that can't be expressed in BPF (e.g., session affinity by application-layer data).

## Combined with Thread-Per-Core

```
Per core:
  ring = io_uring_setup(SINGLE_ISSUER | COOP_TASKRUN | NO_SQARRAY)
  sock = socket(SO_REUSEPORT)
  bind + listen
  multishot ACCEPT → direct descriptors
  multishot RECV per connection → provided buffer ring
  SEND / SEND_ZC for responses
```

This is the zero-contention architecture. Each core is fully independent — no locks, no shared state, no cross-thread anything. The kernel's SO_REUSEPORT hash is the only coordination point.

## BPF Steering Example

For NUMA-aware distribution:

```c
// BPF program attached via SO_ATTACH_REUSEPORT_EBPF
SEC("sk_reuseport")
int numa_steer(struct sk_reuseport_md *ctx) {
    // Route to socket index matching NUMA node of incoming NIC queue
    return bpf_get_numa_node_id();
}
```

## Gotchas

- **Socket migration on thread exit:** If a thread dies, its queued connections are lost (no migration). Use `SO_REUSEPORT_ATTACH_BPF` with failover logic.
- **Uneven distribution:** Hash-based distribution can be uneven with few source IPs. BPF program can implement round-robin or least-connections.
- **RSS alignment:** For best NAPI integration, align SO_REUSEPORT socket count with NIC RSS queue count.
