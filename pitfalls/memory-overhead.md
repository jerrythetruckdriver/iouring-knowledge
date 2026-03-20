# io_uring Memory Overhead Analysis

## Per-Ring Overhead

Every `io_uring_setup()` allocates:

| Component | Size | Formula |
|-----------|------|---------|
| SQ ring | ~`sq_entries * 4` + header | SQ index array (unless NO_SQARRAY) |
| CQ ring | `cq_entries * 16` | 16 bytes per CQE (or 32 with CQE32) |
| SQE array | `sq_entries * 64` | 64 bytes per SQE (or 128 with SQE128) |
| Internal ctx | ~2-4 KB | `struct io_ring_ctx` + ancillary |

### Typical Configurations

| Ring Size | SQ+CQ+SQEs | Notes |
|-----------|------------|-------|
| 32 entries | ~6 KB | Minimal, good for embedded |
| 256 entries | ~44 KB | Default for most apps |
| 1024 entries | ~164 KB | High-throughput server |
| 4096 entries | ~640 KB | Maximum practical |
| 32768 entries | ~5 MB | CQ can be up to 2x SQ |

CQ is always at least 2x SQ entries. With `IORING_SETUP_CQSIZE`, you can set CQ independently.

### NO_SQARRAY Savings

`IORING_SETUP_NO_SQARRAY` (6.7) removes the `sq_entries * 4` byte SQ index array. For 4096 entries, that's 16 KB saved. Small, but it also simplifies the submission path.

### Mixed CQE/SQE Savings

Without mixed mode: need CQE32 ring even if only 1% of ops need 32-byte CQEs.

| Ring Size | CQE16 only | CQE32 only | CQE_MIXED (1% 32b) |
|-----------|-----------|-----------|-------------------|
| 1024 CQEs | 16 KB | 32 KB | ~16.2 KB |
| 4096 CQEs | 64 KB | 128 KB | ~64.6 KB |

Same logic applies to SQE_MIXED (6.19) for the rare 128-byte SQE.

## Per-Request Overhead

Each in-flight request (`struct io_kiocb`) costs approximately:

| Component | Size |
|-----------|------|
| `io_kiocb` base | ~200-280 bytes |
| Operation-specific data | 0-64 bytes |
| Linked chain pointers | 8-16 bytes per link |

This is kernel-allocated per SQE submission. A 1024-entry ring with all slots filled: ~280 KB kernel memory.

## Registered Resources

### Registered Buffers

```
Per buffer: struct bio_vec (24 bytes) + page pinning overhead
1024 registered buffers × 4KB each: 4 MB pinned + ~24 KB metadata
```

Pinned pages are the real cost. Registered buffers lock physical pages, counted against `RLIMIT_MEMLOCK`.

### Registered Files

```
Per file: ~8 bytes (file pointer in fixed table)
1024 registered files: ~8 KB
```

Cheap. The benefit is avoiding `fget()`/`fput()` atomic refcount ops per I/O.

### Provided Buffer Rings

```
Ring header: 16 bytes
Per buffer entry: 16 bytes (struct io_uring_buf)
Ring page: mmap'd, one page minimum

256-entry buffer ring: 4 KB (one page)
4096-entry buffer ring: 64 KB
```

Plus the actual buffer memory you allocate.

## SQPOLL Thread

One kernel thread per ring (or shared via `IORING_SETUP_ATTACH_WQ`). Costs:

- Kernel stack: 8-16 KB
- Continuous CPU when active (spins for `sq_thread_idle` ms)
- task_struct: ~6 KB

## io-wq Worker Pool

Per bounded worker: ~24 KB (kernel stack + task_struct).
Workers spawn on demand, limited by `IORING_REGISTER_IOWQ_MAX_WORKERS` and `RLIMIT_NPROC`.

Default max bounded workers: `min(RLIMIT_NPROC, online_cpus * 4)`.

## Total Memory Budget

### Minimal Setup (embedded / low-throughput)
```
Ring (32 entries):    ~6 KB
No registered resources
No SQPOLL
Total: ~8 KB kernel + mmap
```

### Typical Server
```
Ring (256 entries):   ~44 KB
Registered files (64): ~0.5 KB
Provided buffer ring (256 × 4KB): 1 MB + 4 KB ring
SQPOLL thread:        ~16 KB
Total: ~1.1 MB
```

### High-Performance (thread-per-core, 8 cores)
```
8 rings (1024 entries): 8 × 164 KB = 1.3 MB
Registered files (1024 per ring): ~64 KB
Registered buffers (256 × 64KB per ring): 128 MB pinned + 48 KB metadata
8 SQPOLL threads: 128 KB
Provided buffer rings: 8 × 1 MB = 8 MB
Total: ~137 MB (dominated by registered buffers)
```

## Key Insight

Ring overhead is negligible. Registered buffer memory dominates. Size your registered buffers carefully — that's where the real memory goes.

Use `IORING_REGISTER_RESIZE_RINGS` (6.13) to start small and grow. Don't pre-allocate for peak load.
