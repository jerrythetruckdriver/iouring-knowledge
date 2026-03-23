# io_uring in Kafka Clients

## librdkafka (C/C++)

The dominant Kafka client library. Uses `poll()` + internal thread pool for I/O. No io_uring support.

**Architecture**: librdkafka spawns internal broker threads, each managing TCP connections to Kafka brokers via blocking sockets + poll(). The user-facing API is callback-based but the internal I/O is traditional.

**Why no io_uring**: librdkafka is cross-platform (Linux, macOS, Windows, FreeBSD, Solaris, AIX). Adding io_uring would only benefit Linux and isn't worth the maintenance cost for a library where network I/O isn't the bottleneck — batch processing, compression, and serialization dominate.

**Could it help?** Marginally. Kafka's wire protocol is batch-oriented: produce/fetch requests contain many messages. The per-request syscall overhead is already amortized across thousands of messages. io_uring's batch submission wouldn't change much.

## sarama (Go)

Go's native Kafka client. Uses Go's `net` package (goroutines + netpoller = epoll under the hood). No io_uring, can't use it from Go without cgo.

## confluent-kafka-go

Go wrapper around librdkafka via cgo. Inherits librdkafka's poll-based I/O.

## kafka-clients (Java)

The official Apache Kafka client (Java). Uses `java.nio.channels.Selector` (epoll on Linux). No io_uring path. Netty's io_uring transport isn't used here.

## franz-go

Pure Go Kafka client. Same Go runtime limitation — epoll via netpoller.

## rdkafka-rs (Rust)

Rust wrapper around librdkafka. Inherits C I/O model.

## Where io_uring Would Actually Matter

Not in the Kafka client. In the **Kafka broker** or in the **application around the client**.

### Kafka Broker (Java)
Uses NIO for network I/O and `FileChannel` for log segment writes. The log write path (sequential append + fsync) is where io_uring could help:
- Linked write + fsync for durability
- Registered files for hot log segments
- IOPOLL for NVMe-backed log directories

But the broker is Java. Until Project Loom + io_uring integration happens (it hasn't), this is theoretical.

### Application Side
If your application consumes from Kafka and writes to a database/file, io_uring helps on the output side:

```
Kafka consumer (librdkafka/poll) → your code → io_uring writes to disk/network
```

The Kafka consumption is rarely the bottleneck. Processing and downstream I/O are.

## Redpanda: The io_uring Kafka

Redpanda is a Kafka-compatible streaming platform written in C++ on Seastar. It uses io_uring for:
- All storage I/O (log segments, indexes)
- Network I/O (client connections)
- Thread-per-core architecture with one ring per shard

See [frameworks/redpanda.md](redpanda.md) for details. If you want Kafka with io_uring, use Redpanda instead of trying to add io_uring to Kafka clients.

## Bottom Line

Kafka clients don't need io_uring. The protocol is batch-oriented, syscall overhead is negligible per-message, and most clients are either cross-platform or in languages that can't use io_uring. The io_uring wins are on the broker side (Redpanda proves this) and in the application's downstream I/O.
