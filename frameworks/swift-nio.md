# Swift NIO io_uring Integration

Swift NIO (Apple's server-side networking framework) has io_uring support in mainline, behind the `SWIFTNIO_USE_IO_URING` compile flag.

## Status

- **PR #1804** (April 2021): Initial implementation by @hassila
- **In mainline**: `LinuxUring.swift` (636 lines) + `SelectorUring.swift`
- **Compile flag**: `SWIFTNIO_USE_IO_URING` (opt-in, not default)
- **Multishot**: Optional via `SWIFTNIO_IO_URING_MULTISHOT` flag
- **Platform**: Linux + Android (`#if os(Linux) || os(Android)`)

## Architecture

NIO uses io_uring as a **poll replacement**, not for direct I/O operations:

```
┌─────────────────────────────┐
│  NIO EventLoop              │
│  ┌───────────────────────┐  │
│  │ Selector (URing)      │  │
│  │  io_uring_prep_poll_* │  │
│  │  (readiness notif.)   │  │
│  └───────────────────────┘  │
│  Actual I/O: read()/write() │
│  (standard syscalls)        │
└─────────────────────────────┘
```

**Operations used:**
- `io_uring_prep_poll_add` — register fd interest (POLLIN/POLLOUT/POLLRDHUP)
- `io_uring_prep_poll_remove` — deregister fd
- `io_uring_poll_update` — modify poll mask (multishot mode)

**Not used:** read, write, accept, connect, send, recv — none of the actual I/O.

## Ring Configuration

```swift
let ringEntries: CUnsignedInt = 8192
let cqeMaxCount: UInt32 = 8192
// io_uring_queue_init(ringEntries, &ring, 0)  // no special flags
```

- 8192 SQ entries
- No SQPOLL, no SINGLE_ISSUER, no COOP_TASKRUN
- One ring per EventLoop

## User Data Packing

64-bit user_data packed as:
- 32 bits: file descriptor
- 16 bits: registration ID (truncated from full SelectorRegistrationID)
- 8 bits: event type (poll/pollModify/pollDelete)

## Multishot vs Singleshot

**Singleshot** (default): After each event, re-registers the poll. Simple but more SQEs.

**Multishot** (`SWIFTNIO_IO_URING_MULTISHOT`): One registration, multiple CQEs. Uses `IORING_POLL_ADD_MULTI` and `io_uring_poll_update` for mask changes. Edge-triggered — needs refresh after partial reads.

**EventFD wakeups always singleshot** — certain NIO usage patterns generate excessive wakeups. Singleshot for eventfd is ~30-35% faster on some benchmarks.

## Submission Batching

```swift
// Deferred reregistrations: batch all poll changes, submit once
if deferReregistrations && self.deferredReregistrationsPending {
    self.deferredReregistrationsPending = false
    ring.io_uring_flush()
}
```

The flush loop handles EAGAIN/EBUSY (CQ backpressure), ENOMEM (kernel memory pressure), and partial submissions from failed fds mid-flight.

## Event Merging

CQEs for the same (fd, registrationID) are merged into a single event before propagating up to the Selector. Minimizes work for NIO's channel handling.

## What's Good

- **Proper multishot support** with poll update for mask changes
- **Event merging** reduces per-event overhead
- **Deferred submission batching** — single syscall per event loop tick
- **Graceful fallback** on ECANCELED, EBADF (fd closed while poll pending)

## What's Missing

- **Poll-only** — doesn't use io_uring for actual I/O (read/write/accept/send/recv)
- **No registered files/buffers** — same syscall overhead for data transfer
- **No SQPOLL/COOP_TASKRUN** — missed optimization opportunities
- **Not default** — requires compile flag, most NIO users never enable it

## Why Poll-Only?

NIO's Channel abstraction does I/O in the EventLoop thread synchronously after readiness notification. Replacing that with io_uring submissions would require restructuring the Channel pipeline — a massive API change for marginal gain in most workloads.

The honest take: using io_uring as a poll replacement captures maybe 10-20% of io_uring's potential. The real wins (batched I/O, zero-copy, registered buffers) require architectural changes NIO isn't positioned to make without breaking its API.

## Comparison

| Feature | NIO epoll | NIO io_uring | Full io_uring |
|---------|-----------|-------------|---------------|
| Readiness notification | ✅ | ✅ | ✅ |
| Batched submission | ❌ | ✅ | ✅ |
| Async I/O | ❌ | ❌ | ✅ |
| Zero-copy | ❌ | ❌ | ✅ |
| Registered resources | ❌ | ❌ | ✅ |
| Multishot accept/recv | ❌ | ❌ | ✅ |

## Source

- `Sources/NIOPosix/LinuxUring.swift` — ring wrapper, URingUserData, event processing
- `Sources/NIOPosix/SelectorUring.swift` — Selector backend, register/reregister/deregister, whenReady loop
- PR #1804: github.com/apple/swift-nio/pull/1804
