# Provided Buffer Ring Sizing

Calculator and guidelines for provided buffer ring (pbuf_ring) configuration.

## Parameters

Three decisions: **buffer count**, **buffer size**, **ring count** (buffer groups).

### Buffer Size

Depends on protocol:

| Workload | Buffer Size | Reasoning |
|----------|------------|-----------|
| HTTP headers | 4 KB | Most headers < 2 KB, 4 KB handles 99.9% |
| HTTP bodies | 16-64 KB | Match typical response sizes |
| DNS | 512 B - 4 KB | Most DNS responses < 512 B, EDNS up to 4 KB |
| QUIC/UDP datagrams | 1350-1500 B | MTU-sized |
| Game server | 256-512 B | Small update packets |
| Database wire protocol | 8-32 KB | Row sets, prepared statement results |
| File transfer | 64-256 KB | Throughput-oriented |
| gRPC | 16 KB | Protobuf messages, streaming chunks |

**When in doubt:** 4 KB. It's a page, aligns naturally, works for most protocols.

### Buffer Count

```
buffers_needed = max_concurrent_connections × buffers_per_connection × safety_margin

buffers_per_connection:
  - Request-response (HTTP): 1-2 (one in-flight, one being processed)
  - Streaming (video, bulk transfer): 2-4
  - Multishot recv: 1 per outstanding CQE (kernel holds buffer until app consumes)

safety_margin: 1.5-2x
```

**Ring entries must be power of 2.** Round up.

### Memory Math

```
Total memory = ring_entries × buffer_size + ring_header (page-aligned)

Example: 10K HTTP connections, 4 KB buffers, 2 buffers/conn, 2x safety
  buffers = 10000 × 2 × 2 = 40000 → round to 65536 (next power of 2)
  memory = 65536 × 4096 = 256 MB

Example: DNS server, 512 B buffers, 4096 entries
  memory = 4096 × 512 = 2 MB

Example: Game server, 256 B buffers, 8192 entries
  memory = 8192 × 256 = 2 MB
```

### Multiple Buffer Groups

Split by size class to avoid waste:

```
Group 0: 512 B × 4096  — small messages (headers, control)
Group 1: 4 KB × 2048   — medium messages (request bodies)
Group 2: 64 KB × 256   — large transfers
```

Select group per-SQE via `buf_group`. Common pattern: start with small group, upgrade on short buffer (recv returns full buffer = message was truncated → resubmit with larger group).

## Incremental Consumption (IOU_PBUF_RING_INC, 6.10)

Changes the math entirely. Instead of one buffer per recv completion, the kernel tracks a read pointer within each buffer. Multiple recvs can consume from the same large buffer.

```
Without INC: 10K connections × 4 KB = 40 MB minimum
With INC:    256 × 64 KB = 16 MB (shared, incrementally consumed)
```

**When to use INC:** Many small messages on many connections. The buffer isn't returned until fully consumed, so size buffers large enough that they serve multiple recvs.

**CQE_F_BUF_MORE flag:** Tells you the buffer will produce more completions. Don't recycle until a completion without this flag.

## Monitoring

`IORING_REGISTER_PBUF_STATUS` (opcode 26) returns the buffer group head position:

```
available_buffers = (tail - head) % ring_entries
```

If `available_buffers` is consistently near zero, you're undersized. If it's consistently near `ring_entries`, you're oversized (wasting memory).

## Refill Strategy

Userspace refills by advancing the ring tail. Two approaches:

1. **Eager:** Refill immediately after processing each CQE with a buffer
2. **Batched:** Refill in batches after processing N CQEs

Batched is cheaper — fewer memory barriers. A batch of 16-32 is typical.

```c
// After processing CQEs
io_uring_buf_ring_add(br, buf, buf_len, bid, mask, count++);
// After batch
io_uring_buf_ring_advance(br, count);  // single store-release
```

## Anti-Patterns

- **One massive buffer group for everything.** Split by size class.
- **Exact-fit buffer count.** Always overprovision — kernel won't wait for refill.
- **Non-power-of-2 entries.** Registration fails.
- **Forgetting refill.** Kernel returns `-ENOBUFS` when group is empty, terminating multishot.
- **Using old PROVIDE_BUFFERS API.** Use pbuf_ring. Always.
