# Memcached

## Overview

Memcached has **experimental io_uring support** for its proxy layer. Enabled via `--enable-proxy-uring` at build time and `proxy_uring=yes` at runtime.

Not for the core memcached server (libevent). Just the proxy — the component that routes requests to backend memcached instances.

## Architecture

From `proxy.h`:

```c
#ifdef HAVE_LIBURING
#include <liburing.h>
#define PRING_QUEUE_SQ_ENTRIES 2048
#define PRING_QUEUE_CQ_ENTRIES 16384
#endif
```

One ring per event thread:

```c
struct proxy_event_thread {
    struct io_uring ring;
    proxy_event_t ur_notify_event;    // eventfd for request notifications
    proxy_event_t ur_benotify_event;  // eventfd for backend connections
    eventfd_t event_counter;
    eventfd_t beevent_counter;
    bool use_uring;
};
```

Key design decisions:
- **2048 SQ entries, 16384 CQ entries** — 8:1 CQ:SQ ratio (aggressive, expects burst completions)
- **eventfd bridge** — inter-thread notification from worker threads to event threads
- **Callback-based** — `proxy_event_cb` takes `struct io_uring_cqe *cqe` directly

## What io_uring Handles

The proxy backend connection layer:
- Socket connect to backend memcached instances
- Read/write for proxied requests and responses
- Connection lifecycle management
- Timeout handling

The proxy receives requests from clients (via libevent), routes them to backends (via io_uring), and returns responses.

## Code Structure

Common code path for both modes:

```c
/* Helper routines common to io_uring and libevent modes */
// common code path for io_uring/epoll/tls/etc
```

The proxy abstracts over libevent and io_uring with a callback interface. Backend connection management, request queuing, and state machines are shared. Only the I/O submission and completion paths differ.

## Why Only the Proxy?

The core memcached server uses libevent for client connections. Replacing it with io_uring would mean:
- Rewriting the entire connection handler
- Breaking cross-platform support
- Marginal gains (memcached is memory-bound, not syscall-bound)

The proxy is different. It's a network proxy making many backend connections — exactly where io_uring's batched submission helps. A proxy thread might manage hundreds of backend connections, each doing connect + send request + read response. That's 6+ syscalls per proxied request with epoll, batched into one io_uring_enter with io_uring.

## Configuration

```bash
# Build with io_uring support
./configure --enable-proxy --enable-proxy-uring

# Runtime enable
memcached -o proxy_uring=yes
```

Runtime detection via stats:
```
stats proxy
STAT use_uring true
```

## Status

**Experimental.** The `configure.ac` description literally says "EXPERIMENTAL":

```
AS_HELP_STRING([--enable-proxy-uring], [Enable proxy io_uring code EXPERIMENTAL])
```

The architecture is sound but it's not the default path. Most memcached deployments use the standard libevent mode.

## What They Could Add

- Multishot recv for backend responses (reduce SQE count)
- Provided buffer rings (avoid per-request buffer allocation)
- Registered files for persistent backend connections
- Linked connect+send for new connections
- NAPI busy poll for low-latency proxy use cases

Currently it's basic io_uring: submit individual read/write/connect ops and reap CQEs. The advanced features that would make the proxy significantly faster aren't used yet.
