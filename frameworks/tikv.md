# TiKV

## Overview

TiKV is a distributed key-value store (Rust, PingCAP). Underpins TiDB. Uses RocksDB via `rust-rocksdb` (C FFI wrapper).

**No direct io_uring usage.** Zero io_uring references in the TiKV codebase.

## RocksDB Underneath

TiKV uses RocksDB for storage. RocksDB has production io_uring support (thread-local rings for MultiRead and ReadAsync). Whether TiKV's RocksDB build enables liburing depends on the build configuration.

The `rust-rocksdb` wrapper compiles RocksDB from source. If `liburing` is found during the cmake build step, RocksDB enables io_uring automatically. This means TiKV _might_ be using io_uring for storage reads without any Rust-level code changes.

## Network Layer

TiKV uses gRPC-rs (gRPC C core via FFI) for Raft consensus and client communication. gRPC C core uses epoll. No io_uring path exists.

Network is not the bottleneck for TiKV anyway. Raft consensus latency dominates — the cost of a network round-trip dwarfs syscall overhead.

## Where io_uring Would Help

1. **Scan queries** — RocksDB MultiRead already uses io_uring for batch SST lookups
2. **Compaction** — RocksDB compaction reads/writes could benefit from registered buffers
3. **WAL writes** — linked write+fsync chains for durability

All of these happen inside RocksDB, not in TiKV's Rust code. TiKV doesn't need its own io_uring integration — it inherits whatever RocksDB provides.

## Why No Direct Integration

- **Rust + RocksDB FFI** — adding another FFI layer for io_uring adds complexity
- **gRPC** — locked to epoll via gRPC C core
- **Cross-platform** — TiKV runs on Linux and macOS
- **Raft consensus** — fsync latency and network RTT dominate, not syscall count
- **Complexity** — RocksDB handles the I/O hot path

## Comparison with Other Raft Stores

| Store | Language | io_uring | Notes |
|-------|----------|----------|-------|
| TiKV | Rust | Via RocksDB | No direct usage |
| etcd | Go | No | bbolt mmap, Go runtime |
| CockroachDB | Go | No | Pebble (Go), no cgo |
| FoundationDB | C++ | **Yes** | Boost.Asio + RocksDB |
| Redpanda | C++ | **Yes** | Seastar, thread-per-core |

Pattern: C++ databases with custom I/O stacks adopt io_uring. Rust/Go databases inherit it (at best) via storage engine FFI.
