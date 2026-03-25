# Per-SQE I/O Priority

## Overview

The `ioprio` field in SQE controls I/O scheduling priority. Two uses:
1. **Block I/O priority** — interacts with the I/O scheduler (mq-deadline, BFQ)
2. **Operation-specific flags** — overloaded for accept, recv, poll operations

## Block I/O Priority

For read/write/fsync operations targeting block devices:

```c
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, fd, buf, len, offset);

// IOPRIO_CLASS_RT (real-time), IOPRIO_CLASS_BE (best-effort), IOPRIO_CLASS_IDLE
// Priority level 0-7 within class (0 = highest)
sqe->ioprio = IOPRIO_PRIO_VALUE(IOPRIO_CLASS_BE, 4);
```

### How It Works

The ioprio value follows the same encoding as `ioprio_set(2)`:

```
Bits 13-15: class (1=RT, 2=BE, 3=IDLE)
Bits 0-12:  level (0-7 for RT/BE, 0 for IDLE)
```

The I/O scheduler uses this to order requests:
- **mq-deadline**: RT class gets lower deadline, BE is default, IDLE is last
- **BFQ**: maps to BFQ weight, affects bandwidth allocation
- **none**: ignored (FIFO ordering)

### Practical Impact

With BFQ scheduler:
```c
// High-priority WAL write
sqe->ioprio = IOPRIO_PRIO_VALUE(IOPRIO_CLASS_RT, 0);

// Low-priority compaction read
sqe->ioprio = IOPRIO_PRIO_VALUE(IOPRIO_CLASS_IDLE, 0);
```

With mq-deadline (NVMe default): priority has minimal effect. NVMe hardware queues process requests in parallel regardless of software priority.

With none scheduler: no effect at all.

**Reality check**: on NVMe with `none` scheduler (the common case for io_uring + IOPOLL), ioprio for block I/O does nothing. It matters on spinning disks or when using BFQ.

## Operation-Specific Flags

The ioprio field is overloaded for non-block operations:

### Accept Flags

```c
// Multishot accept
sqe->ioprio = IORING_ACCEPT_MULTISHOT;

// Don't wait if no pending connections
sqe->ioprio = IORING_ACCEPT_DONTWAIT;

// Do initial poll check before accept
sqe->ioprio = IORING_ACCEPT_POLL_FIRST;

// Combine
sqe->ioprio = IORING_ACCEPT_MULTISHOT | IORING_ACCEPT_POLL_FIRST;
```

### Recv/Send Flags

```c
// Multishot recv (with provided buffers)
sqe->ioprio = IORING_RECV_MULTISHOT;

// Bundle recv (batch into provided buffers)
sqe->ioprio = IORING_RECVSEND_BUNDLE;

// Bundle + multishot
sqe->ioprio = IORING_RECV_MULTISHOT | IORING_RECVSEND_BUNDLE;
```

### Poll Flags

```c
// Multishot poll (rearms automatically)
sqe->ioprio = IORING_POLL_ADD_MULTI;

// Update events on existing poll
sqe->ioprio = IORING_POLL_UPDATE_EVENTS;

// Level-triggered (default first CQE is level, subsequent are edge)
sqe->ioprio = IORING_POLL_ADD_LEVEL;
```

## Priority Mixing on Same Ring

You can submit SQEs with different priorities in the same batch:

```c
// WAL write — high priority
struct io_uring_sqe *sqe1 = io_uring_get_sqe(&ring);
io_uring_prep_write(sqe1, wal_fd, data, len, offset);
sqe1->ioprio = IOPRIO_PRIO_VALUE(IOPRIO_CLASS_RT, 0);

// Compaction read — idle priority
struct io_uring_sqe *sqe2 = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe2, sst_fd, buf, rlen, roff);
sqe2->ioprio = IOPRIO_PRIO_VALUE(IOPRIO_CLASS_IDLE, 0);

// Both submitted in one io_uring_submit()
```

The I/O scheduler differentiates them. Useful for databases mixing latency-sensitive writes with background reads.

## cgroup Interaction

If the process is in an io.latency or io.cost cgroup, ioprio interacts with the cgroup controller. RT-class I/O from a throttled cgroup may still be throttled. The cgroup controller wins.

## Summary

| Context | ioprio Meaning | Impact |
|---------|---------------|--------|
| Read/write on NVMe (none scheduler) | Block I/O priority | None |
| Read/write on HDD (BFQ) | Block I/O priority | Significant |
| Accept | MULTISHOT, DONTWAIT, POLL_FIRST | Feature flags |
| Recv/Send | MULTISHOT, BUNDLE | Feature flags |
| Poll | MULTI, UPDATE, LEVEL | Feature flags |

Don't set ioprio for block I/O unless you know your scheduler respects it. Always set it for multishot/bundle operations — those flags are required, not optional.
