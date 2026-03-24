# Connection Pooling Patterns

io_uring can optimize connection pool implementations — both database pools and HTTP/2 multiplexing.

## Database Connection Pool

Traditional connection pool: N idle connections, check out on request, return on completion. io_uring changes the I/O pattern.

### Idle Connection Monitoring

```c
// Monitor all idle connections with multishot poll
for (int i = 0; i < pool_size; i++) {
    struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
    io_uring_prep_poll_multishot(sqe, pool[i].fd, POLLIN | POLLHUP | POLLERR);
    sqe->user_data = POOL_MONITOR | i;
}
io_uring_submit(&ring);
```

Detects:
- **Server-initiated close** (POLLHUP) — remove stale connection
- **Unexpected data** (POLLIN on idle connection) — protocol error or notification
- **Error** (POLLERR) — network failure

### Health Check Pattern

```c
// Periodic health check: linked write + read with timeout
struct io_uring_sqe *sqe;

// 1. Send ping
sqe = io_uring_get_sqe(&ring);
io_uring_prep_send(sqe, conn_fd, ping_msg, ping_len, 0);
sqe->flags |= IOSQE_IO_LINK;

// 2. Read response (linked)
sqe = io_uring_get_sqe(&ring);
io_uring_prep_recv(sqe, conn_fd, buf, buf_len, 0);
sqe->flags |= IOSQE_IO_LINK;

// 3. Timeout (cancel chain if too slow)
sqe = io_uring_get_sqe(&ring);
io_uring_prep_link_timeout(sqe, &timeout_1s, 0);
```

### Batch Query Pattern

```c
// Send queries to multiple pool connections simultaneously
for (int i = 0; i < batch_size; i++) {
    conn_t *c = checkout_connection(pool);
    
    struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
    io_uring_prep_send(sqe, c->fd, queries[i].data, queries[i].len, 0);
    sqe->user_data = QUERY_SEND | i;
}
io_uring_submit(&ring);  // One syscall for all sends
```

vs traditional: N send() syscalls.

## HTTP/2 Multiplexing

HTTP/2 multiplexes streams over one TCP connection. io_uring patterns:

### Frame Coalescing

```c
// Batch multiple HTTP/2 frames into vectored send
struct iovec iovs[MAX_FRAMES];
int frame_count = 0;

// Collect pending frames across streams
for (stream = streams_head; stream; stream = stream->next) {
    if (stream->pending_data) {
        iovs[frame_count].iov_base = stream->frame_buf;
        iovs[frame_count].iov_len = stream->frame_len;
        frame_count++;
    }
}

// Single writev via io_uring
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_writev(sqe, conn_fd, iovs, frame_count, 0);
```

### Multishot Recv for Frame Parsing

```c
// Continuous frame reception
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_recv_multishot(sqe, conn_fd, 0, 0);  // provided buffers
sqe->buf_group = HTTP2_BUF_GROUP;
sqe->flags |= IOSQE_BUFFER_SELECT;
```

Each CQE delivers a chunk of the HTTP/2 byte stream. Parse frame headers, demux to streams.

## Pool-per-Thread with MSG_RING

Thread-per-core architecture: each thread has its own connection pool and ring.

```
Thread 0: Ring₀ + Pool₀ (connections 0-N)
Thread 1: Ring₁ + Pool₁ (connections N-2N)
...

// Request arrives on Thread 0 but needs Thread 2's connection
// Use MSG_RING to dispatch
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring0);
io_uring_prep_msg_ring(sqe, ring2_fd, QUERY_REQUEST, query_id, 0);
```

## Connection Establishment

Batch connection creation for pool warmup:

```c
// Create N connections simultaneously
for (int i = 0; i < pool_size; i++) {
    int fd = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK, 0);
    
    struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
    io_uring_prep_connect(sqe, fd, &server_addr, sizeof(server_addr));
    sqe->user_data = POOL_CONNECT | i;
}
io_uring_submit(&ring);  // All connects in flight simultaneously
```

vs traditional: sequential connect() calls or select/poll loop.

## Reconnection

```c
// On connection failure: linked socket + connect + send health check
struct io_uring_sqe *sqe;

sqe = io_uring_get_sqe(&ring);
io_uring_prep_socket(sqe, AF_INET, SOCK_STREAM, 0, IORING_FILE_INDEX_ALLOC);
sqe->flags |= IOSQE_IO_LINK;

sqe = io_uring_get_sqe(&ring);
io_uring_prep_connect(sqe, 0, &server_addr, sizeof(server_addr));
sqe->flags |= IOSQE_FIXED_FILE | IOSQE_IO_LINK;

sqe = io_uring_get_sqe(&ring);
io_uring_prep_send(sqe, 0, health_check, health_len, 0);
sqe->flags |= IOSQE_FIXED_FILE;
```

Three operations, zero syscalls until submit.

## Performance Characteristics

| Operation | Traditional | io_uring |
|-----------|-----------|----------|
| Pool warmup (N connects) | N+ syscalls | 1 submit |
| Batch query (N sends) | N syscalls | 1 submit |
| Idle monitoring | epoll_ctl per fd | multishot poll |
| Health check | send + recv + timeout | linked chain |
| Reconnect | socket + connect | linked chain |

## PgBouncer / Connection Pooler Use Case

Connection poolers like PgBouncer sit between clients and databases, managing a pool of backend connections. They're I/O multiplexers — exactly where io_uring shines:

- **Client side**: multishot accept + multishot recv for client connections
- **Backend side**: batched query forwarding via send, multishot recv for results
- **Idle monitoring**: multishot poll on all idle backend connections
- **Timeout management**: link timeouts on query forwarding

PgBouncer is C, single-threaded, uses libevent (epoll). A rewrite on io_uring would eliminate the libevent dependency and reduce syscall overhead significantly for high-connection-count deployments.
