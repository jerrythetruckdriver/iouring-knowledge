# Batch Connect Patterns

Establishing many outbound connections simultaneously — crawlers, health checkers, load testers, connection pool warmup.

## Basic Batch Connect

```c
for (int i = 0; i < batch_size; i++) {
    struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
    io_uring_prep_socket(sqe, AF_INET, SOCK_STREAM, 0,
                         IORING_FILE_INDEX_ALLOC);
    sqe->user_data = ENCODE_OP(OP_SOCKET, i);
}
io_uring_submit(&ring);

// On socket CQE: chain connect
void on_socket_complete(int idx, int fixed_fd) {
    struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
    io_uring_prep_connect(sqe, fixed_fd, &addrs[idx], addrlen);
    sqe->flags |= IOSQE_FIXED_FILE;
    sqe->user_data = ENCODE_OP(OP_CONNECT, idx);
}
```

## Linked Socket+Connect

Single SQE chain per connection:

```c
for (int i = 0; i < batch_size; i++) {
    // SQE 1: socket
    struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
    io_uring_prep_socket(sqe, AF_INET, SOCK_STREAM, 0,
                         IORING_FILE_INDEX_ALLOC);
    sqe->flags |= IOSQE_IO_LINK;
    sqe->user_data = ENCODE_OP(OP_SOCKET, i);

    // SQE 2: connect (uses result of socket)
    sqe = io_uring_get_sqe(&ring);
    io_uring_prep_connect(sqe, 0, &addrs[i], addrlen);
    sqe->flags |= IOSQE_FIXED_FILE;
    sqe->user_data = ENCODE_OP(OP_CONNECT, i);

    // SQE 3: timeout (cancel if connect takes too long)
    sqe = io_uring_get_sqe(&ring);
    io_uring_prep_link_timeout(sqe, &connect_timeout, 0);
    sqe->user_data = ENCODE_OP(OP_TIMEOUT, i);
}
io_uring_submit(&ring); // One submit for N*3 SQEs
```

**Note**: Linked connect uses `file_index` 0 — the result of the preceding socket op fills the direct descriptor slot, and connect references it. The kernel resolves the dependency within the link chain.

## Connection Pool Warmup

```c
// Warm up pool: establish N connections, then start accepting work
int pending = 0;
int established = 0;

// Phase 1: submit all connection attempts
for (int i = 0; i < pool_size; i++) {
    submit_socket_connect_chain(i);
    pending++;
}
io_uring_submit(&ring);

// Phase 2: process completions
while (established < pool_size) {
    io_uring_cqe *cqe;
    io_uring_wait_cqe(&ring, &cqe);

    int op = DECODE_OP(cqe->user_data);
    int idx = DECODE_IDX(cqe->user_data);

    if (op == OP_CONNECT && cqe->res == 0) {
        pool_add(idx);
        established++;
    } else if (op == OP_CONNECT && cqe->res < 0) {
        // Retry with backoff or skip
        retry_connect(idx);
    }
    io_uring_cqe_seen(&ring, cqe);
}
```

## Rate-Limited Connect

Don't hammer a target with 10,000 simultaneous connects:

```c
// Sliding window: max N concurrent connect attempts
#define MAX_CONCURRENT 64

int in_flight = 0;
int next_conn = 0;

while (next_conn < total_connections || in_flight > 0) {
    // Submit up to window
    while (in_flight < MAX_CONCURRENT && next_conn < total_connections) {
        submit_connect(next_conn++);
        in_flight++;
    }
    io_uring_submit(&ring);

    // Drain completions
    io_uring_cqe *cqe;
    io_uring_wait_cqe(&ring, &cqe);
    handle_completion(cqe);
    in_flight--;
    io_uring_cqe_seen(&ring, cqe);
}
```

## DNS + Connect Pipeline

Resolve then connect, all async:

```c
// Use c-ares for DNS resolution, POLL_ADD to watch resolver fd
// On resolution complete, submit socket+connect chain
void on_dns_resolved(int idx, struct sockaddr *addr) {
    memcpy(&addrs[idx], addr, sizeof(*addr));
    submit_socket_connect_chain(idx);
    io_uring_submit(&ring);
}
```

## Syscall Comparison

| Approach | Syscalls for 1000 connections |
|---|---|
| Blocking `connect()` | 3,000+ (socket + connect + fcntl per conn) |
| epoll + non-blocking | 4,000+ (socket + fcntl + connect + epoll_ctl) |
| io_uring batch | 1-2 (io_uring_enter for submit + wait) |

## Gotchas

1. **SQ depth**: 1000 linked chains = 3000 SQEs. Size your ring accordingly, or batch in waves.
2. **Connect timeouts**: Use link timeout, not a separate cancel loop.
3. **EINPROGRESS**: io_uring handles this internally — you get the final result in CQE.
4. **Direct descriptors**: `IORING_FILE_INDEX_ALLOC` auto-allocates. Pre-allocate with `REGISTER_FILES` for large pools.
5. **SO_REUSEADDR**: Set via `URING_CMD` (6.11) or before submitting the socket op.
