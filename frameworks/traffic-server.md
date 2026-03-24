# Apache Traffic Server

## Status: Production io_uring support

Apache Traffic Server (ATS) has a dedicated io_uring subsystem. One of the few production HTTP proxy/cache servers with real io_uring integration.

## Architecture

```
ATS Event System
├── EThread (event thread, one per core)
│   ├── EventLoop (epoll-based for networking)
│   └── IOUringContext (thread-local io_uring ring)
│       ├── Ring (liburing, configurable depth)
│       ├── eventfd bridge → EventLoop
│       └── Probe (feature detection)
└── AIO subsystem
    ├── Thread-pool AIO (legacy)
    └── io_uring AIO (TS_USE_LINUX_IO_URING)
```

### Source Analysis (`src/iocore/io_uring/`)

**IOUringContext** — thread-local ring wrapper:

```c
// include/iocore/io_uring/IO_URING.h
struct IOUringConfig {
    int queue_entries = 32;   // SQ depth
    int sq_poll_ms    = 0;    // SQPOLL idle timeout (0 = disabled)
    int attach_wq     = 0;    // Share io-wq workers
    int wq_bounded    = 0;    // Bounded worker limit
    int wq_unbounded  = 0;    // Unbounded worker limit
};
```

Key design decisions:
- **Thread-local rings**: `thread_local IOUringContext threadContext` — one ring per EThread
- **SQPOLL support**: Optional via config (`sq_poll_ms > 0`)
- **Shared work queue**: `IORING_SETUP_ATTACH_WQ` — worker threads shared across rings
- **Feature probing**: `io_uring_get_probe_ring()` at init, `supports_op()` checks before use
- **eventfd bridge**: Registers eventfd with ring, bridges CQE notifications into epoll event loop
- **Metrics**: `proxy.process.io_uring.submitted` / `proxy.process.io_uring.completed` counters

### Completion Model

```c
// IOUringCompletionHandler — virtual interface
class IOUringCompletionHandler {
    virtual void handle_complete(io_uring_cqe *) = 0;
};

// SQE submission with handler
io_uring_sqe *sqe = ctx->next_sqe(handler);
// If SQ full, auto-submits and retries
```

The `next_sqe()` method handles SQ backpressure: if the SQ is full, it submits pending SQEs and tries again.

### Event Loop Integration

```
IOUringEventIO::start()
    → register_eventfd() — creates eventfd, registers with ring
    → start_common(loop, eventfd, EVENTIO_READ)

IOUringEventIO::process_event()
    → service() — peek all CQEs, dispatch to handlers
    → read(evfd) — drain eventfd counter
```

Epoll watches the eventfd. When CQEs arrive, epoll wakes, drains all completions, dispatches to handlers.

## What ATS Uses io_uring For

- **Disk AIO**: Cache reads/writes (primary use case)
- **Configurable per-deploy**: `TS_USE_LINUX_IO_URING` compile flag

Networking still uses epoll. io_uring handles the storage path — similar to how Seastar uses it.

## What ATS Doesn't Use

- Registered buffers
- Registered files
- Multishot operations
- Linked operations
- Zero-copy send/recv
- COOP_TASKRUN / SINGLE_ISSUER

The implementation is conservative but correct. Thread-local rings with shared work queue, feature probing, and clean eventfd bridging.

## Configuration

```yaml
# records.yaml (ATS config)
io_uring:
  queue_entries: 32
  sq_poll_ms: 0        # 0 = no SQPOLL
  attach_wq: 1         # Share worker threads
  wq_bounded: 0        # Default worker limits
  wq_unbounded: 0
```

## Significance

ATS is one of the few production L7 proxy/cache servers with io_uring. Most competitors (Nginx, HAProxy, Envoy, Caddy, Traefik) don't have it. The cache workload is the right place: random read I/O on disk, where io_uring's async submission and batching reduce latency.

## Source

- `src/iocore/io_uring/io_uring.cc` — ring lifecycle, submit, service, worker config
- `src/iocore/io_uring/IOUringEventIO.cc` — eventfd bridge to event loop
- `include/iocore/io_uring/IO_URING.h` — IOUringContext class, IOUringConfig struct
- `src/iocore/aio/AIO.cc` — AIO subsystem with `TS_USE_LINUX_IO_URING` integration
