# Multishot Operations

One SQE, many CQEs. The most important pattern for high-performance io_uring.

## Why Multishot Matters

Traditional model: submit accept SQE → get CQE → submit another accept SQE. That's one syscall round-trip per connection.

Multishot: submit once → get CQEs forever. One SQE handles thousands of connections.

## Available Multishot Operations

| Operation | Flag/Opcode | Since |
|-----------|------------|-------|
| Accept | `IORING_ACCEPT_MULTISHOT` (ioprio) | 5.19 |
| Recv | `IORING_RECV_MULTISHOT` (ioprio) | 6.0 |
| Poll | `IORING_POLL_ADD_MULTI` (len) | 5.13 |
| Read | `IORING_OP_READ_MULTISHOT` | 6.6 |
| Timeout | `IORING_TIMEOUT_MULTISHOT` | 6.7 |

## The CQE Protocol

Every CQE from a multishot operation has `IORING_CQE_F_MORE` set — except the last one. When you see `IORING_CQE_F_MORE` cleared, the multishot is done. You need to resubmit.

```c
if (!(cqe->flags & IORING_CQE_F_MORE)) {
    // Multishot terminated — resubmit if needed
    rearm_multishot(ring, user_data);
}
```

Common termination causes:
- Error (CQE `res < 0`)
- Buffer pool exhausted
- Cancellation

## Multishot Accept Pattern

```c
struct io_uring_sqe *sqe = io_uring_get_sqe(ring);
io_uring_prep_multishot_accept(sqe, listen_fd, NULL, NULL, 0);
sqe->user_data = ACCEPT_TAG;

// Optional: direct fd allocation
sqe->file_index = IORING_FILE_INDEX_ALLOC;
```

## Multishot Recv + Provided Buffers

This is the killer combo for network servers:

```c
struct io_uring_sqe *sqe = io_uring_get_sqe(ring);
io_uring_prep_recv_multishot(sqe, client_fd, NULL, 0, 0);
sqe->buf_group = BUF_GROUP_ID;
sqe->flags |= IOSQE_BUFFER_SELECT;
sqe->user_data = make_user_data(client_fd, RECV_TAG);
```

Processing:
```c
int buf_id = cqe->flags >> IORING_CQE_BUFFER_SHIFT;
int len = cqe->res;
void *data = get_buffer(buf_id);
// process data[0..len]
return_buffer(buf_ring, buf_id);  // return to provided buffer ring
```

## CQE Overflow with Multishot

Multishot ops can generate CQEs faster than you consume them. Size your CQ ring accordingly:

```c
params.cq_entries = sq_entries * 8;  // generous for multishot
params.flags |= IORING_SETUP_CQSIZE;
```

If CQ overflows, multishot ops may be terminated. Check `IORING_SQ_CQ_OVERFLOW` flag.

## Skip CQE on Success

With `IOSQE_CQE_SKIP_SUCCESS`, successful completions don't produce CQEs. Useful for fire-and-forget sends paired with multishot recv. Only error CQEs show up.

Doesn't work with multishot operations themselves (they need CQEs to deliver results). Use it on the send side.
