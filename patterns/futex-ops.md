# io_uring Futex Operations

Added in kernel 6.7. Three opcodes for async futex operations.

## Opcodes

| Opcode | Value | Description |
|--------|-------|-------------|
| `IORING_OP_FUTEX_WAIT` | 49 | Async futex wait |
| `IORING_OP_FUTEX_WAKE` | 50 | Async futex wake |
| `IORING_OP_FUTEX_WAITV` | 51 | Async vectored futex wait (multiple futexes) |

## Why This Exists

Before this, futex operations were synchronous. If you had an io_uring event loop and needed to wait on a futex, you had two bad options:

1. Block a thread on `futex(FUTEX_WAIT)` — defeats the purpose
2. Use `IORING_OP_POLL_ADD` on an eventfd paired with the futex — extra fd, extra complexity

Now you submit a futex wait as an SQE. When the futex fires, you get a CQE. The event loop never blocks.

## SQE Layout

```c
// FUTEX_WAIT
sqe->opcode = IORING_OP_FUTEX_WAIT;
sqe->fd = 0;                    // unused
sqe->addr = (uintptr_t)futex;   // pointer to futex word
sqe->addr2 = expected_val;      // value to compare against
sqe->futex_flags = FUTEX2_SIZE_U32;  // futex2 flags
sqe->len = FUTEX_WAIT;          // futex op (can use FUTEX_WAIT_BITSET)

// FUTEX_WAKE
sqe->opcode = IORING_OP_FUTEX_WAKE;
sqe->addr = (uintptr_t)futex;
sqe->len = FUTEX_WAKE;
sqe->addr2 = nr_wake;           // how many waiters to wake

// FUTEX_WAITV — wait on multiple futexes
sqe->opcode = IORING_OP_FUTEX_WAITV;
sqe->addr = (uintptr_t)futex_waitv_array;
sqe->len = nr_futexes;
```

## Use Cases

### Async Mutex/Condvar in Event Loops

```c
// Submit futex wait as part of event loop
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_futex_wait(sqe, &mutex_word, expected,
                         FUTEX_BITSET_MATCH_ANY,
                         FUTEX2_SIZE_U32, 0);
sqe->user_data = MUTEX_WAIT_TAG;

// Later, wake from another SQE in same ring
sqe = io_uring_get_sqe(&ring);
io_uring_prep_futex_wake(sqe, &mutex_word, 1,
                         FUTEX_BITSET_MATCH_ANY,
                         FUTEX2_SIZE_U32, 0);
```

### Cross-Thread Signaling Without Eventfd

Before: thread A writes to eventfd, io_uring polls it.
Now: thread A does `FUTEX_WAKE` (or regular syscall wake), io_uring's `FUTEX_WAIT` completes.

One less fd. One less indirection.

### FUTEX_WAITV — Wait on Multiple Conditions

The vectored variant is particularly powerful. Wait for *any* of N futexes to fire, get one CQE. Useful for:
- Waiting on multiple locks simultaneously
- Implementing select()-like semantics for userspace synchronization primitives
- Game engines waiting on multiple worker thread completion signals

## Flags

Uses `futex2` flag space via `sqe->futex_flags`:

| Flag | Value | Description |
|------|-------|-------------|
| `FUTEX2_SIZE_U8` | 0x00 | 8-bit futex |
| `FUTEX2_SIZE_U16` | 0x01 | 16-bit futex |
| `FUTEX2_SIZE_U32` | 0x02 | 32-bit futex (most common) |
| `FUTEX2_SIZE_U64` | 0x03 | 64-bit futex |
| `FUTEX2_NUMA` | 0x04 | NUMA-aware (6.16+) |

## Kernel Version Requirements

- **6.7**: `FUTEX_WAIT`, `FUTEX_WAKE`, `FUTEX_WAITV` landed
- **6.16**: `FUTEX2_NUMA` and `FUTEX2_MPOL` support for NUMA-aware futexes

## Gotchas

1. **Not the same as futex(2) syscall flags** — uses `futex2` flag space. `FUTEX2_SIZE_U32` is required for standard 32-bit futexes.
2. **CQE result**: 0 on success for WAIT, number woken for WAKE. Negative errno on failure.
3. **EAGAIN**: Returned if the futex value doesn't match expected at submission time (just like regular futex WAIT).
4. **Cancellation**: Supports `IORING_OP_ASYNC_CANCEL`. Cancel a pending futex wait cleanly.
