# Completion Ordering Guarantees

io_uring completions (CQEs) are **unordered by default**. Understanding what's ordered, what's not, and how to enforce ordering is critical for correctness.

## Default: No Ordering

CQEs appear in the CQ ring in the order they complete, not the order they were submitted. Two reads submitted as SQE[0] and SQE[1] may complete as CQE[1] then CQE[0] if the second read's data was cached.

```
Submit: READ_A, READ_B, WRITE_C
Complete: WRITE_C, READ_A, READ_B  (any permutation possible)
```

This is fundamental to io_uring's performance. If completions were ordered, the kernel would have to serialize — defeating the purpose.

## What IS Ordered

### Within a Link Chain

Linked operations (IOSQE_IO_LINK / IOSQE_IO_HARDLINK) execute sequentially. CQEs for a link chain appear in submission order **relative to each other**, but may be interleaved with CQEs from other requests.

```
Submit: [WRITE → FSYNC] (linked), READ_X (independent)
Possible CQE order: READ_X, WRITE, FSYNC     ✅
Possible CQE order: WRITE, READ_X, FSYNC     ✅
Impossible:         FSYNC, WRITE, READ_X      ❌ (FSYNC before WRITE)
```

Soft links (IOSQE_IO_LINK): next op starts only after previous succeeds. Chain aborts on failure (subsequent ops get -ECANCELED).

Hard links (IOSQE_IO_HARDLINK): next op starts regardless of previous result. No abort on failure.

### IOSQE_IO_DRAIN

A drain barrier. The drained SQE waits for all previously submitted SQEs to complete before executing.

```
Submit: A, B, C (drain), D
Execution: A and B run concurrently
           C waits for A and B to complete
           D waits for C to complete
```

Expensive — serializes the entire ring. Use sparingly.

### Multishot CQEs

Multiple CQEs from a single multishot SQE (accept, recv, poll, read) are ordered relative to each other. You won't get recv data out of order from a single multishot recv.

### CQE_SKIP Gaps (Mixed CQE Mode)

With IORING_SETUP_CQE_MIXED (6.18), a 32-byte CQE that can't fit before ring wrap generates a SKIP CQE to fill the gap. SKIPs maintain ring contiguity but carry no user data. Always check IORING_CQE_F_SKIP.

## What's NOT Ordered

| Scenario | Ordered? |
|----------|----------|
| Independent SQEs | ❌ Any completion order |
| Same-fd reads at different offsets | ❌ |
| Reads to different fds | ❌ |
| SQEs across different `io_uring_submit()` calls | ❌ |
| Multiple multishot streams (different SQEs) | ❌ relative to each other |
| MSG_RING from different source rings | ❌ |
| SQPOLL-submitted vs user-submitted | ❌ |

## Enforcing Order

### Method 1: Link Chains (Best)

```c
// Write then fsync — guaranteed order
sqe1 = io_uring_get_sqe(&ring);
io_uring_prep_write(sqe1, fd, data, len, off);
sqe1->flags |= IOSQE_IO_LINK;

sqe2 = io_uring_get_sqe(&ring);
io_uring_prep_fsync(sqe2, fd, 0);
```

### Method 2: Sequential Submission (Suboptimal)

Submit, wait for CQE, submit next. Works but defeats batching.

### Method 3: user_data Sequencing

Encode sequence numbers in user_data. Reorder in userspace on completion. No kernel overhead.

```c
// Encode: upper 32 bits = request type, lower 32 bits = sequence
sqe->user_data = ((uint64_t)REQ_TYPE << 32) | sequence_number;
```

### Method 4: DRAIN (Nuclear Option)

```c
sqe->flags |= IOSQE_IO_DRAIN;
```

Serializes everything. Kills throughput.

## CQ Ring Ordering vs Logical Ordering

The CQ ring is a FIFO. CQEs are consumed in ring order (head→tail). But that ring order reflects completion order, not submission order. Don't confuse "I read CQEs sequentially" with "operations completed in the order I submitted them."

## Practical Patterns

**WAL write path:** Link write→fsync. Guarantees data is on disk before acknowledging.

**Request-response:** Encode request ID in user_data. Match CQEs to requests. No ordering needed.

**Pipeline stages:** Use link chains within stages, independent submission across stages.

**Event loop:** Process all available CQEs in a batch. Don't assume anything about their order. Use user_data to dispatch to the right handler.

## The Rule

If you need operation A to complete before operation B starts: **link them**. For everything else, design your application to handle completions in any order. That's how you get io_uring's full throughput.
