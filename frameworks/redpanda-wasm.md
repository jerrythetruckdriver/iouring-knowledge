# Redpanda Wasm Transforms and io_uring

## TL;DR

Redpanda's Wasm data transforms run in V8 isolates. They don't interact with io_uring directly — Seastar's reactor handles all I/O. The Wasm layer is pure computation on data that's already in memory.

## Architecture

```
Redpanda core (Seastar, C++)
  ├── Storage (io_uring via Seastar reactor)
  ├── Networking (epoll via Seastar reactor)  
  └── Wasm Transform Engine
      └── V8 isolate per transform
          └── User's JS/Wasm function
              └── Operates on in-memory records only
```

The Wasm transform engine:
1. Seastar reads records from source topic (io_uring for storage I/O)
2. Records are deserialized into Seastar memory
3. V8 isolate executes user transform function on the records
4. Transformed records are written to output topic (io_uring for storage I/O)

## io_uring's Role

io_uring handles the I/O bookending the Wasm execution:

- **Input:** Reading source topic partitions from disk (io_uring read with O_DIRECT)
- **Output:** Writing transformed records to output topic (io_uring write + fdatasync)
- **WAL:** Raft log entries for the transform output

The Wasm function itself never touches io_uring. It receives deserialized records and returns transformed records. No syscalls, no I/O, no ring access.

## Why This Matters

This is the correct architecture. You don't give untrusted user code access to io_uring rings:

1. **Security** — Wasm sandbox can't escape to make syscalls
2. **Performance** — Seastar's thread-per-core model owns the rings. One ring per shard, no contention.
3. **Simplicity** — Wasm transforms are pure functions. I/O complexity stays in the engine.

## Comparison with Other Wasm+io_uring Approaches

| System | Wasm role | io_uring role |
|--------|-----------|---------------|
| Redpanda transforms | Computation on records | Storage I/O for input/output |
| Cloudflare Workers | Full request handling | Not used (epoll) |
| Fastly Compute@Edge | Full request handling | Not used (epoll) |
| Hypothetical io_uring Wasm | ? | ? |

Nobody exposes io_uring to Wasm guests. The WASI async I/O proposal might eventually bridge this gap, but that's theoretical. See [wasi-webassembly.md](../patterns/wasi-webassembly.md).
