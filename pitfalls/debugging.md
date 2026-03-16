# Debugging io_uring

io_uring is invisible to `strace`. Your I/O requests don't map to syscalls — they're submitted through shared memory and reaped from a completion ring. You need different tools.

## Tracepoints

io_uring ships static tracepoints in the kernel. These are your primary visibility tool.

### Discovering Tracepoints

```bash
# perf
sudo perf list 'io_uring:*'

# bpftrace
sudo bpftrace -l 'tracepoint:io_uring:*'

# tracefs
ls /sys/kernel/tracing/events/io_uring/
```

### Key Tracepoints

| Tracepoint | Fires When |
|------------|------------|
| `io_uring:io_uring_create` | Ring initialized |
| `io_uring:io_uring_register` | `io_uring_register()` called |
| `io_uring:io_uring_submit_sqe` | SQE consumed from submission queue |
| `io_uring:io_uring_queue_async_work` | Request dispatched to worker thread |
| `io_uring:io_uring_poll_arm` | Async poll registered (non-blocking path) |
| `io_uring:io_uring_poll_wake` | Polled fd became ready |
| `io_uring:io_uring_complete` | CQE posted to completion queue |
| `io_uring:io_uring_defer` | Request deferred (dependency or drain) |
| `io_uring:io_uring_link` | Linked SQE detected |
| `io_uring:io_uring_fail_link` | Linked request chain broken |
| `io_uring:io_uring_cqring_wait` | Entering CQ wait |
| `io_uring:io_uring_task_add` | Completion added to task work |
| `io_uring:io_uring_task_run` | Task work completion running |

### Request Flow

```
submit_sqe ──▶ [poll_arm] ──▶ [poll_wake] ──▶ complete
                  │
                  ▼
          [queue_async_work] ──▶ complete
```

Requests either take the fast poll path (non-blocking attempt → poll for readiness) or get dispatched to a worker thread.

## bpftrace Recipes

### Count operations by opcode

```bash
sudo bpftrace -e '
tracepoint:io_uring:io_uring_submit_sqe {
    @ops[args->opcode] = count();
}
interval:s:5 { print(@); clear(@); }'
```

### Track submission-to-completion latency

```bash
sudo bpftrace -e '
tracepoint:io_uring:io_uring_submit_sqe {
    @start[args->req] = nsecs;
}
tracepoint:io_uring:io_uring_complete /@start[args->req]/ {
    @latency_us = hist((nsecs - @start[args->req]) / 1000);
    delete(@start[args->req]);
}'
```

### Detect worker thread spawning

```bash
sudo bpftrace --btf -e '
kretprobe:create_io_thread {
    @spawn[retval] = count();
}
interval:s:1 { print(@spawn); clear(@spawn); }'
```

If you see `-11` (`-EAGAIN`) counts, io_uring is failing to spawn workers — check `RLIMIT_NPROC` or cgroup `pids.max`.

### Monitor CQ overflow

```bash
sudo bpftrace -e '
tracepoint:io_uring:io_uring_complete /args->cflags & 0x8000/ {
    printf("CQ overflow! req=%p\n", args->req);
    @overflow = count();
}'
```

## perf stat

Quick sanity check on tracepoint hit counts:

```bash
# Are requests going async or polling?
sudo perf stat -a -e \
    io_uring:io_uring_submit_sqe,\
    io_uring:io_uring_poll_arm,\
    io_uring:io_uring_queue_async_work,\
    io_uring:io_uring_complete \
    -- sleep 5
```

If `poll_arm` ≈ `submit_sqe`, requests are taking the fast non-blocking path. If `queue_async_work` is high, you're hitting the worker pool.

## ftrace

For lightweight kernel tracing without BPF:

```bash
cd /sys/kernel/tracing

# Enable io_uring events
echo 1 > events/io_uring/enable

# Read trace
cat trace_pipe | head -100

# Disable
echo 0 > events/io_uring/enable
```

Filter by specific event:

```bash
echo 1 > events/io_uring/io_uring_complete/enable
cat trace_pipe
```

## /proc Inspection

### Worker threads

```bash
# Count io_uring worker threads
pstree -pt <pid> | grep iou-wrk | wc -l

# Thread names show owner
# iou-wrk-<tid> — worker owned by thread <tid>
```

### Ring instances

```bash
# Each io_uring instance shows as anon_inode:[io_uring]
ls -la /proc/<pid>/fd | grep io_uring
```

### fdinfo

```bash
# Detailed ring stats (kernel 5.18+)
cat /proc/<pid>/fdinfo/<ring_fd>
```

Shows: SqEntries, CqEntries, SqMask, SqThread (if SQPOLL), UserFiles, UserBufs, and more.

## Common Debug Scenarios

### "My requests aren't completing"

1. Check if they're in the poll path: trace `io_uring_poll_arm`
2. Check if they're stuck in worker threads: count `iou-wrk-*` threads
3. Check if CQ is overflowing: look for `CQE_F_MORE` without consumption

### "High CPU in kernel"

1. Worker spawn failure: trace `create_io_thread` return values
2. SQPOLL spinning: check if SQ thread is consuming CPU without submissions
3. CQ overflow handling: kernel retries posting CQEs

### "Requests returning unexpected errors"

CQE `res` field contains negative errno:

```bash
sudo bpftrace -e '
tracepoint:io_uring:io_uring_complete /args->res < 0/ {
    printf("error: op=%d res=%d (-%s)\n",
        args->opcode, args->res,
        args->res == -11 ? "EAGAIN" :
        args->res == -4 ? "EINTR" :
        args->res == -125 ? "ECANCELED" : "other");
    @errors[args->res] = count();
}'
```

### "SQPOLL thread not submitting"

```bash
# Check if SQ thread is idle (needs wakeup)
sudo bpftrace -e '
kprobe:io_sq_thread {
    @sq_thread_runs = count();
}
interval:s:1 { print(@sq_thread_runs); clear(@sq_thread_runs); }'
```

## Performance Profiling

### Syscall overhead

```bash
# Count io_uring_enter calls
sudo perf stat -e syscalls:sys_enter_io_uring_enter -- sleep 5
```

If this is high, you're not batching effectively. Goal: minimize enter() calls per I/O operation.

### Ring utilization

```bash
sudo bpftrace -e '
tracepoint:io_uring:io_uring_submit_sqe {
    @batch_size = lhist(args->sq_entries, 0, 256, 8);
}'
```

## Tools

| Tool | Best For |
|------|----------|
| `bpftrace` | Quick one-liners, latency histograms |
| `perf stat` | Aggregate tracepoint counts |
| `perf record` + `perf script` | Full event timelines |
| `ftrace` | Lightweight, no BPF dependency |
| `/proc/<pid>/fdinfo` | Ring configuration inspection |
| `pstree -pt` | Worker thread topology |
