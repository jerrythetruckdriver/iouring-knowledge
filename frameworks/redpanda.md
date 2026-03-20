# Redpanda

## What It Is

Kafka-compatible streaming platform. C++, built on Seastar (same foundation as ScyllaDB). Thread-per-core, one io_uring ring per shard.

## Architecture

Redpanda inherits Seastar's I/O architecture wholesale:

- **Thread-per-core**: One shard per CPU core, no shared mutable state
- **One ring per shard**: Seastar's reactor owns the io_uring instance
- **Direct I/O**: O_DIRECT for all data path operations (WAL, segments)
- **No page cache**: Application-managed buffers, registered with io_uring

## io_uring Usage

Redpanda's I/O goes through Seastar's `io_queue`, which uses io_uring for:

1. **Segment writes**: Kafka topic partition segments written via io_uring WRITE_FIXED
2. **Segment reads**: Consumer fetches via io_uring READ_FIXED
3. **fsync**: Durability guarantees via linked write+fsync chains
4. **Network I/O**: Accept, recv, send through Seastar's network stack (io_uring-backed since Seastar adopted it)

## vs ScyllaDB

Same Seastar foundation, different I/O patterns:

| Aspect | ScyllaDB | Redpanda |
|---|---|---|
| Primary I/O | Random reads (LSM) | Sequential writes (log) |
| IOPOLL benefit | High (random NVMe) | Lower (sequential) |
| NVMe passthrough | Yes (explored) | Less relevant |
| Write pattern | Compaction-heavy | Append-only segments |

Redpanda's sequential write pattern means epoll vs io_uring matters less for raw throughput. The win is in batching and reduced syscall overhead at high message rates.

## Performance Characteristics

- **Latency**: Thread-per-core eliminates cross-thread coordination. P99 latency is predictable.
- **Throughput**: At high partition counts (thousands), io_uring's batch submission reduces per-partition overhead vs individual write() calls.
- **CPU efficiency**: SQPOLL + registered buffers keeps the data path in userspace.

## What's Notable

Redpanda proves that io_uring + thread-per-core works for **streaming workloads**, not just database workloads. The append-only log pattern maps cleanly to linked write+fsync chains.

The Seastar dependency means Redpanda automatically benefits from Seastar's io_uring improvements without touching Redpanda's I/O code directly.

## Source

- GitHub: `redpanda-data/redpanda` (C++, Bazel build)
- I/O layer: inherited from Seastar's `reactor` and `io_queue`
- Not a custom io_uring integration — it's Seastar's
