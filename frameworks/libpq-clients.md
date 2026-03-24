# PostgreSQL Client Libraries and io_uring

## libpq (C)

No io_uring. libpq uses blocking or non-blocking sockets with poll/select. The wire protocol (pgproto3) is inherently request-response — send query, wait for results. io_uring wouldn't help the protocol roundtrip.

libpq's non-blocking mode (`PQsetnonblocking`) returns control between send/recv, letting the caller integrate with their own event loop. That event loop *could* use io_uring (via POLL_ADD on the socket fd), but libpq itself doesn't care.

## Where io_uring Actually Helps

### Connection Pooling (PgBouncer, Odyssey, Supavisor)

Connection poolers multiplex many client connections to fewer server connections. This is a networking workload — accept, recv, send, across hundreds or thousands of fds. Classic io_uring territory:

- **Multishot accept** for incoming clients
- **Multishot recv** on client and server sockets
- **Bundle send** for batching responses
- **Direct descriptors** for zero fd table overhead

No major pooler uses io_uring yet. PgBouncer uses libevent (epoll). Odyssey uses a custom event loop (epoll). Supavisor is Elixir (BEAM VM, epoll).

### Bulk Data Loading (COPY)

`COPY FROM` streams data into PostgreSQL. The bottleneck is usually parsing and WAL writes, not socket I/O. But for very fast networks (100GbE), batching COPY data with io_uring send could reduce syscall overhead.

### Large Result Sets

Streaming large query results (`FETCH` with cursors, or just a big SELECT) generates lots of recv calls. Multishot recv eliminates per-row syscall overhead.

## Client Libraries by Language

| Library | Language | io_uring | Event Loop |
|---------|----------|----------|------------|
| libpq | C | No | poll/select |
| asyncpg | Python | No | uvloop/asyncio |
| node-postgres | Node.js | No | libuv (epoll) |
| pq-sys/tokio-postgres | Rust | No | tokio (epoll) |
| pgx/jackc | Go | No | net poller (epoll) |
| npgsql | .NET | No | System.IO.Pipelines |
| JDBC (pgjdbc) | Java | No | NIO (epoll) |

None use io_uring. The wire protocol's request-response nature means syscall overhead isn't the bottleneck — network latency and query execution time are.

## The Only Path: Server-Side

The real io_uring work is happening *inside* PostgreSQL (Andres Freund's async I/O subsystem), not in clients. Server-side benefits:

- WAL write+fsync chains (linked ops)
- Buffer pool prefetch (batched reads)
- Checkpoint parallelism

Client libraries don't need io_uring because they're not doing enough I/O to benefit. A single connection does maybe a few hundred roundtrips per second — each is one send + one recv. epoll handles this trivially.

## When a Client Would Benefit

Theoretical scenario: a proxy or middleware that handles 100K+ concurrent PostgreSQL connections, doing protocol-level routing. At that scale, io_uring's multishot accept/recv and reduced syscall overhead matter. But at that point you're building a pooler, not a client library.
