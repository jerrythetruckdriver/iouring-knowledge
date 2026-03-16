# Glommio

Thread-per-core async runtime for Rust, built on io_uring. Created by Glauber Costa (Datadog).

## Architecture

- One io_uring ring per CPU core
- No cross-thread work stealing (unlike Tokio)
- Cooperative task scheduling within each core
- All I/O is io_uring — no fallback to epoll

## io_uring Features Used

- `SQPOLL` mode (optional, for storage workloads)
- Registered buffers for DMA-aligned I/O
- Provided buffer rings for networking
- Multishot accept and recv
- `SINGLE_ISSUER` + `COOP_TASKRUN`
- Direct file descriptors

## Design Philosophy

Thread-per-core eliminates synchronization. Each core has:
- Its own io_uring instance
- Its own task queue
- Its own memory allocator
- Its own connection handling

Cross-core communication via shared memory channels, not locks.

## When to Use

- High-throughput network services
- Storage-heavy workloads (databases, caches)
- When you want maximum io_uring utilization
- When Tokio's work-stealing overhead matters

## Trade-offs

- No work stealing = uneven load distribution possible
- Requires understanding thread-per-core model
- Smaller ecosystem than Tokio
- Linux-only (io_uring dependency)

## Source

- Repository: https://github.com/DataDog/glommio
- License: Apache-2.0/MIT
