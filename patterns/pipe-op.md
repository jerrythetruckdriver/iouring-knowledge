# IORING_OP_PIPE

Opcode 60 (6.14). Creates a pipe asynchronously.

## SQE Layout

```c
sqe->opcode  = IORING_OP_PIPE;
sqe->fd      = -1;        // unused
sqe->len     = 0;         // reserved
sqe->pipe_flags = flags;  // O_NONBLOCK, O_CLOEXEC, etc.
```

Uses `IORING_FILE_INDEX_ALLOC` for direct descriptor allocation — both ends go into the fixed file table.

## CQE Result

`cqe->res` returns the read-end fd (or fixed index). Write-end is `res + 1` when using fixed files.

## Why It Matters

Before 6.14, creating a pipe required `pipe2()` — a synchronous syscall that breaks the io_uring submission chain. Now you can:

1. Create pipe (PIPE)
2. Splice data into it (SPLICE)  
3. Send pipe fd to another ring (MSG_RING + SEND_FD)

All without leaving the ring.

## Pattern: Async Pipe + Splice

```c
// SQE 1: create pipe with fixed descriptors
sqe = io_uring_get_sqe(ring);
io_uring_prep_pipe(sqe, O_NONBLOCK);
sqe->file_index = IORING_FILE_INDEX_ALLOC;
sqe->flags |= IOSQE_IO_LINK;

// SQE 2: splice into pipe (chained)
sqe = io_uring_get_sqe(ring);
io_uring_prep_splice(sqe, src_fd, -1, /* pipe write fd from alloc */, -1, len, 0);
```

## When to Use

- Building data pipelines entirely within io_uring
- Avoiding syscall gaps in linked operation chains
- Proxy/relay patterns where pipe+splice beats userspace copy
