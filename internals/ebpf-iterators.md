# eBPF Iterators for io_uring Introspection

## What They Are

BPF iterators (`bpf_iter`) provide a way to iterate over kernel data structures from BPF programs and dump results to userspace via `bpf_seq_printf`. For io_uring, this means non-intrusive ring inspection without modifying the target process.

## Current State (6.19)

There is **no dedicated `bpf_iter__io_uring` iterator type** in the kernel as of 6.19. io_uring introspection via BPF relies on:

1. **Task iterators** (`bpf_iter__task`) — walk all tasks, inspect their io_uring context via BTF
2. **FD iterators** (`bpf_iter__task_file`) — find io_uring ring fds, read ring state
3. **Tracepoints** — `io_uring:*` tracepoint family for dynamic observation
4. **kprobes/fentry** — attach to io_uring internal functions

## Task Iterator Approach

```c
// Attach to bpf_iter__task, filter for tasks with io_uring_task
SEC("iter/task")
int dump_io_uring_tasks(struct bpf_iter__task *ctx)
{
    struct task_struct *task = ctx->task;
    if (!task)
        return 0;

    struct io_uring_task *tctx = task->io_uring;
    if (!tctx)
        return 0;

    // Read ring count, inflight requests, etc. via BTF
    BPF_SEQ_PRINTF(ctx->meta->seq, "pid=%d inflight=%d\n",
                   task->pid, tctx->inflight_tracked);
    return 0;
}
```

This works but only gives you task-level metadata. Ring-internal state (SQ/CQ positions, pending ops) requires deeper BTF access.

## FD Iterator + BTF

More useful: iterate file descriptors, identify io_uring rings, read their state:

```c
SEC("iter/task_file")
int dump_rings(struct bpf_iter__task_file *ctx)
{
    struct file *file = ctx->file;
    if (!file)
        return 0;

    // Check if this fd is an io_uring instance
    // file->f_op == &io_uring_fops
    struct io_ring_ctx *ring_ctx; // access via file->private_data + BTF
    // Read sq.sqe_tail, cq.cqe_head, etc.
}
```

**Caveat**: Kernel struct layouts change between versions. BTF makes this portable, but io_ring_ctx is not a stable ABI.

## Tracepoint-Based Introspection

More practical for production. The `io_uring:*` tracepoints provide:

| Tracepoint | What It Shows |
|---|---|
| `io_uring_create` | Ring creation (entries, flags) |
| `io_uring_submit_req` | Each SQE submitted (opcode, fd, user_data) |
| `io_uring_complete` | Each CQE posted (res, cflags) |
| `io_uring_queue_async_work` | Request punted to io-wq |
| `io_uring_poll_arm` | Poll armed for request |
| `io_uring_cqe_overflow` | CQ overflow event |

BPF programs attached to these tracepoints can build a live model of ring state.

## What's Missing

- **No `bpf_iter__io_uring_ring`** — can't iterate all rings on the system directly
- **No `bpf_iter__io_uring_request`** — can't iterate in-flight requests
- **No stable io_ring_ctx kfuncs** — no blessed BPF helper to read ring state safely

## /proc Alternative

For quick inspection without BPF:

```bash
# Per-ring info via fdinfo
cat /proc/<pid>/fdinfo/<ring_fd>
# Shows: SqEntries, CqEntries, SqHead, SqTail, CqHead, CqTail, etc.
```

## Trajectory

The natural next step is a `bpf_iter__io_uring` that iterates all io_ring_ctx instances on the system, similar to how `bpf_iter__tcp` iterates sockets. Jens Axboe has expressed interest but it's not prioritized — `/proc/fdinfo` and tracepoints cover most debugging needs.

For production monitoring, tracepoints + BPF maps (counting ops, tracking latency histograms) is the proven approach. Full ring iteration is a debugging luxury.
