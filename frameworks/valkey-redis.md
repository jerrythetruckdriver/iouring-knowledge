# Valkey / Redis — io_uring Adoption

## Status: Active Development (Valkey), Stalled (Redis)

Valkey forked from Redis in March 2024. Both are single-threaded event loop architectures using `ae` (Antirez's Event library) over epoll/kqueue. io_uring integration is happening in Valkey, not Redis.

## Valkey io_uring PRs

### PR #112 — Batch Client Writes via io_uring

Batches pending client write responses through io_uring instead of individual `write()` syscalls. Single-threaded, no io-wq workers — the improvement comes purely from syscall batching.

**Results (single CPU):**
- IPC improvement: 1.16 → 1.23 insn/cycle (+6%)
- Syscall reduction: significant (batch N writes into 1 io_uring_enter)
- No additional CPU cores consumed

### PR #750 — AOF Persistence via io_uring

Replaces synchronous `write()`+`fsync()` for AOF (Append Only File) with io_uring async writes. Critical path for `appendfsync always` mode where every command must hit disk.

**Published benchmarks (Ubuntu, kernel 6.5, SATA SSD, Xeon Gold 6152):**

| Mode | Baseline (QPS) | io_uring (QPS) | Improvement |
|------|----------------|----------------|-------------|
| Single thread | 48,847 | 63,131 | **+29.2%** |
| 4 threads | 59,992 | 72,724 | **+21.2%** |

Config: `appendfsync always`, 100-byte values, 5M operations.

### PR #1784 — Bio Thread Disk Sync (Alternative Approach)

Chose background thread over io_uring for replica RDB download. Rationale: single-connection workload doesn't benefit from io_uring batching. Bio thread gave 80-95% improvement for replica sync.

This is the right call — io_uring wins on batching, not on single sequential I/O.

## Redis

Redis has open issues (#9440, #9441, #10880, #10881, #14644) discussing io_uring but no merged PRs. The project considered and rejected io_uring multiple times. Post-fork, development energy went to Valkey.

## Why io_uring Fits (and Doesn't)

**Where it helps:**
- AOF writes: batch write+fsync reduces latency on `appendfsync always`
- Client response writes: batch N pending responses into one submission
- RDB save: async disk I/O during background save

**Where it doesn't:**
- Core event loop: ae is already efficient for socket polling (epoll is fine here)
- Command processing: CPU-bound, not I/O-bound
- Single-connection workloads: no batching opportunity

**Architectural constraint:** Valkey is fundamentally single-threaded for command execution. io_uring helps at the edges (disk I/O, response batching) but can't change the core model. The real wins come from reducing syscall overhead on the hot path.

## The Pattern

Single-threaded servers benefit from io_uring in two specific ways:
1. **Syscall batching** — N client writes → 1 io_uring_enter
2. **Async disk I/O** — fsync/write without blocking the event loop

They don't benefit from io_uring networking (multishot accept/recv) because their event loop already handles that efficiently.

## Sources

- [Valkey PR #112](https://github.com/valkey-io/valkey/pull/112) — batch writes, perf stat data
- [Valkey PR #750](https://github.com/valkey-io/valkey/pull/750) — AOF io_uring, benchmark data
- [Valkey PR #1784](https://github.com/valkey-io/valkey/pull/1784) — Bio thread (chose over io_uring for replica sync)
