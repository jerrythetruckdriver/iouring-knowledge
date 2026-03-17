# Performance Tuning

How to get every nanosecond out of io_uring. No guessing — just configuration knobs and their tradeoffs.

## Ring Sizing

Default SQ/CQ sizes are powers of 2. CQ defaults to 2× SQ entries.

### SQ Ring

| Workload | Recommended SQ | Why |
|----------|---------------|-----|
| Single connection | 16–32 | Minimal overhead |
| Server (1K conns) | 128–256 | Enough to batch without CQ pressure |
| High-throughput storage | 256–1024 | Amortize syscall overhead |
| SQPOLL | 1024–4096 | Keep the poll thread fed |

Oversizing wastes memory. Each SQE is 64 bytes (128 with `SQE128`). A 4096-entry ring = 256KB.

### CQ Ring

Set with `IORING_SETUP_CQSIZE`. Default is 2× SQ entries. Bump it if:
- Multishot ops generate bursts of CQEs
- Processing is slower than completion rate
- You're seeing `IORING_SQ_CQ_OVERFLOW`

Each CQE is 16 bytes (32 with `CQE32`). Cheap to overprovision. 4× or 8× SQ is fine for bursty workloads.

### Runtime Resize (6.13+)

`IORING_REGISTER_RESIZE_RINGS` lets you grow/shrink without teardown. Start small, grow when backpressure appears:

```c
struct io_uring_params p = { .sq_entries = 64, .cq_entries = 256 };
// ... later, under load:
io_uring_register_resize_rings(ring, 512, 2048);
```

## Batch Strategies

### Submission Batching

The golden rule: **submit many, enter once**.

```c
// Bad: submit one at a time
for (int i = 0; i < N; i++) {
    prep_sqe(ring, i);
    io_uring_submit(ring);  // syscall per op
}

// Good: batch submit
for (int i = 0; i < N; i++) {
    prep_sqe(ring, i);
}
io_uring_submit(ring);  // one syscall for N ops
```

Cost of `io_uring_enter()` is ~200-400ns regardless of batch size. Submitting 1 vs 32 SQEs costs nearly the same.

### Completion Batching

Don't process CQEs one at a time. Drain the CQ ring in a loop:

```c
unsigned head;
struct io_uring_cqe *cqe;
unsigned completed = 0;

io_uring_for_each_cqe(ring, head, cqe) {
    handle_completion(cqe);
    completed++;
}
io_uring_cq_advance(ring, completed);
```

### min_wait_usec (6.13+)

With registered wait, `min_wait_usec` tells the kernel: "don't wake me until this many microseconds have passed, even if a CQE is ready." Trades latency for throughput:

```c
struct io_uring_reg_wait w = {
    .min_wait_usec = 100,  // batch CQEs for 100µs
};
```

Good for high-throughput storage. Bad for latency-sensitive networking.

## Setup Flag Combinations

### Maximum Throughput (Storage)
```c
IORING_SETUP_SQPOLL |
IORING_SETUP_IOPOLL |
IORING_SETUP_NO_SQARRAY |
IORING_SETUP_SINGLE_ISSUER
```
Zero syscalls, hardware poll, no indirection. Needs O_DIRECT + NVMe.

### Maximum Efficiency (Network Server)
```c
IORING_SETUP_COOP_TASKRUN |
IORING_SETUP_TASKRUN_FLAG |
IORING_SETUP_SINGLE_ISSUER |
IORING_SETUP_DEFER_TASKRUN |
IORING_SETUP_NO_SQARRAY
```
No IPIs, deferred task work, single-threaded event loop. The "thread-per-core" setup.

### Balanced (General Purpose)
```c
IORING_SETUP_COOP_TASKRUN |
IORING_SETUP_SINGLE_ISSUER |
IORING_SETUP_NO_SQARRAY
```
Safe defaults. No SQPOLL overhead, no IOPOLL restrictions, still gets IPI reduction.

## Registered Resources

### Why They Matter

Unregistered: every operation does `fdget()` + `fdput()` + buffer import. Registered: kernel keeps pre-mapped references.

| Resource | Registration Cost | Per-Op Savings |
|----------|-----------------|----------------|
| Files | ~100ns per file (once) | ~50-100ns per op |
| Buffers | ~500ns per buffer (once) | ~100-200ns per op |
| Ring fd | Negligible | ~20ns per submit |

### Fixed File Table Best Practices

- Use sparse registration (`IORING_RSRC_REGISTER_SPARSE`)
- Pre-allocate for expected max connections
- Use `IORING_FILE_INDEX_ALLOC` for automatic slot management
- Use `IORING_REGISTER_FILE_ALLOC_RANGE` to limit auto-alloc range

## CQ Overflow Handling

When CQ ring is full:

1. **Detection**: Check `IORING_SQ_CQ_OVERFLOW` in `sq->flags`
2. **Recovery**: Call `io_uring_enter(IORING_GETEVENTS)` to flush overflow list
3. **Prevention**: Size CQ ring 4-8× SQ entries for bursty workloads

With `NODROP` feature (all modern kernels): submissions block rather than drop CQEs. Without it: CQEs silently vanish. Check `IORING_FEAT_NODROP` at setup time.

### CQE Skip

`IOSQE_CQE_SKIP_SUCCESS` suppresses CQEs for successful operations. Massive CQ pressure reduction for fire-and-forget patterns:

```c
// Don't generate CQE unless this fails
sqe->flags |= IOSQE_CQE_SKIP_SUCCESS;
```

Use for: send completions you don't need to track, NOP padding, buffer registration ops.

## NAPI Busy Poll

For network-heavy workloads:

```c
struct io_uring_napi napi = {
    .busy_poll_to = 50,       // µs to busy-poll before sleeping
    .prefer_busy_poll = 1,
};
io_uring_register_napi(ring, &napi);
```

Keeps the NIC RX queue hot. Trades CPU for latency. Only worth it at >100K packets/sec.

## Memory Layout

Pin the ring and SQE/CQE arrays to the right NUMA node. The kernel `mmap()`s them from the io_uring fd — use `mbind()` or `set_mempolicy()` before the mmap, or use `IORING_SETUP_NO_MMAP` to provide your own pre-allocated, NUMA-local memory.

## Profiling

1. **Tracepoints**: `io_uring:io_uring_submit_req`, `io_uring:io_uring_complete` — measure per-op latency
2. **bpftrace**: Histogram of completion times by opcode
3. **perf stat**: Look at `context-switches` and `cpu-migrations` — these kill throughput
4. **CQ overflow counter**: If this is non-zero, your CQ is undersized

## Anti-Patterns

| Pattern | Problem | Fix |
|---------|---------|-----|
| Submit one SQE per `io_uring_enter()` | Wastes syscall overhead | Batch |
| CQ ring same size as SQ | Overflow under multishot | 4× CQ |
| SQPOLL for light workloads | Wastes a CPU core | Only use at >50K ops/sec |
| Not using `COOP_TASKRUN` | Unnecessary IPIs | Always set it (5.19+) |
| Registered 0 resources | Per-op overhead | Register files + buffers |
| Ignoring `CQ_OVERFLOW` | Silent performance degradation | Monitor and resize |
