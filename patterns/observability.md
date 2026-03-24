# io_uring Observability

## TL;DR

No native Prometheus exporter. Build your own from `/proc`, tracepoints, and REGISTER_PBUF_STATUS. The kernel gives you the data — you need to export it.

## Data Sources

### /proc/\<pid\>/fdinfo/\<ring_fd\>

```bash
cat /proc/1234/fdinfo/5
# IoUring:
#   UserFiles:    256
#   UserBufs:     0
#   PollList:     12
#   SqEntries:    1024
#   CqEntries:    2048
#   SubQueueLen:  0
#   ...
```

Available fields: ring config, registered resources, inflight count. Parse and export.

### tracepoints (io_uring:*)

```bash
# List available tracepoints
perf list 'io_uring:*'

# Key tracepoints:
io_uring:io_uring_submit_req    # SQE submitted
io_uring:io_uring_complete      # CQE posted
io_uring:io_uring_cqe_overflow  # CQ overflow (critical)
io_uring:io_uring_queue_async_work  # Offloaded to io-wq
io_uring:io_uring_poll_arm      # Poll armed
io_uring:io_uring_task_work_run # Task work executed
```

### REGISTER_PBUF_STATUS

```c
struct io_uring_buf_status bs = { .buf_group = group_id };
io_uring_register(ring_fd, IORING_REGISTER_PBUF_STATUS, &bs, 1);
// bs.head now contains current consumption position
// available = ring_entries - (tail - bs.head)
```

### io-wq worker monitoring

```bash
# Worker threads visible in /proc
ps -eLf | grep 'iou-wrk'

# Count per process
ls /proc/<pid>/task/ | wc -l  # includes io-wq workers
```

## Metrics to Export

### Ring-level (per ring instance)

| Metric | Source | Type |
|--------|--------|------|
| `iouring_sq_entries` | fdinfo | Gauge |
| `iouring_cq_entries` | fdinfo | Gauge |
| `iouring_inflight` | fdinfo | Gauge |
| `iouring_registered_files` | fdinfo | Gauge |
| `iouring_registered_buffers` | fdinfo | Gauge |
| `iouring_poll_list_len` | fdinfo | Gauge |

### Operation-level (via tracepoints or app instrumentation)

| Metric | Source | Type |
|--------|--------|------|
| `iouring_submissions_total` | tracepoint / app counter | Counter |
| `iouring_completions_total` | tracepoint / app counter | Counter |
| `iouring_cq_overflows_total` | tracepoint / SQ flags | Counter |
| `iouring_async_offloads_total` | tracepoint | Counter |
| `iouring_op_latency_seconds` | app instrumentation | Histogram |

### Buffer-level

| Metric | Source | Type |
|--------|--------|------|
| `iouring_pbuf_available` | REGISTER_PBUF_STATUS | Gauge |
| `iouring_pbuf_exhaustion_total` | app (ENOBUFS count) | Counter |

### System-level

| Metric | Source | Type |
|--------|--------|------|
| `iouring_memlock_bytes` | /proc/\<pid\>/status VmLck | Gauge |
| `iouring_iowq_workers` | ps / /proc/\<pid\>/task | Gauge |

## bpftrace-based Exporter Pattern

```bash
#!/usr/bin/env bpftrace

// Count submissions by opcode
tracepoint:io_uring:io_uring_submit_req
{
    @submissions[args->opcode] = count();
}

// Track CQ overflow events
tracepoint:io_uring:io_uring_cqe_overflow
{
    @overflows = count();
}

// Latency histogram (submit → complete)
tracepoint:io_uring:io_uring_submit_req
{
    @start[args->req] = nsecs;
}

tracepoint:io_uring:io_uring_complete
{
    if (@start[args->req]) {
        @latency_us = hist((nsecs - @start[args->req]) / 1000);
        delete(@start[args->req]);
    }
}
```

Pipe output to a Prometheus textfile collector or push gateway.

## Application-level Instrumentation

Better than tracepoints for production (lower overhead):

```c
// Track in your event loop
struct ring_metrics {
    uint64_t submissions;
    uint64_t completions;
    uint64_t cq_overflows;
    uint64_t short_reads;
    uint64_t async_offloads;
    uint64_t buffer_exhaustions;
};

// After io_uring_submit()
metrics.submissions += submitted;

// After io_uring_peek_cqe() / io_uring_wait_cqe()
metrics.completions++;
if (ring->sq.kflags && (*ring->sq.kflags & IORING_SQ_CQ_OVERFLOW))
    metrics.cq_overflows++;
```

## Tracing Integration

For distributed tracing (OpenTelemetry, Jaeger):

- Encode trace context in `user_data` (64 bits available)
- On CQE completion, extract trace ID and record span
- For linked chains, the first SQE's user_data carries the trace context

**Practical limit:** 64-bit `user_data` isn't enough for a full trace ID (128 bits). Use an index into a per-ring trace context table.

## What Doesn't Exist

- No built-in `/sys/kernel/debug/io_uring/` stats directory
- No BPF iterator specifically for io_uring rings
- No standard Prometheus exporter (yet)
- No OpenTelemetry io_uring instrumentation library

Build it yourself. The data is there.
