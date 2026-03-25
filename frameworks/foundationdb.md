# FoundationDB

## Overview

FoundationDB uses io_uring in **two separate layers**:

1. **Networking** — Boost.Asio with `BOOST_ASIO_HAS_IO_URING=1` and `BOOST_ASIO_DISABLE_EPOLL=1`
2. **Storage** — RocksDB backend with optional `WITH_LIBURING`

This makes FDB one of the few distributed databases running io_uring for both network and disk I/O.

## Boost.Asio io_uring Backend

From `fdbserver/CMakeLists.txt`:

```cmake
target_compile_definitions(fdbserver PRIVATE
  BOOST_ASIO_HAS_IO_URING=1
  BOOST_ASIO_DISABLE_EPOLL=1)
```

They explicitly **disable epoll** and force io_uring. Bold move.

Boost.Asio's io_uring backend replaces the epoll reactor with an io_uring submission queue. All socket operations (connect, read, write, accept) go through the ring instead of epoll_wait + read/write syscalls.

What you get:
- Batched submission of network operations
- No epoll_ctl syscalls for fd registration
- Reduced syscall count for the actor model's many small messages

## RocksDB Storage Layer

Optional but recommended. Controlled by `WITH_LIBURING` cmake flag:

```cmake
if(WITH_ROCKSDB)
  if(WITH_LIBURING)
    find_package(uring REQUIRED)
  endif()
endif()
```

When enabled, RocksDB uses thread-local io_uring rings for:
- **MultiRead** — batch SST random reads (the big win for LSM engines)
- **ReadAsync** — prefetch during iteration
- Poll/AbortIO for lifecycle management

## Architecture

FDB's actor model (Flow language) generates coroutine-like state machines. Each actor yields at I/O boundaries. With Boost.Asio's io_uring backend, these yields become SQE submissions rather than epoll registration + syscall pairs.

The combination matters:
- **Network**: actor messages between nodes → Boost.Asio → io_uring
- **Storage**: key-value reads → RocksDB → io_uring
- **Both paths** through the same subsystem on Linux

## What They Don't Use

- No SQPOLL (actor model doesn't benefit — already batching)
- No registered buffers (Boost.Asio manages its own buffer lifecycle)
- No zero-copy networking (Boost.Asio abstraction doesn't expose it)
- No direct descriptors (Boost.Asio manages fds)

## Why It Works for FDB

FDB's actor model generates thousands of small messages. The syscall overhead of epoll_ctl + read + write per message adds up. io_uring batches these naturally.

But the Boost.Asio abstraction means they capture maybe 30-40% of io_uring's potential. No multishot, no provided buffers, no zero-copy. It's io_uring as a better epoll, not io_uring as a new I/O paradigm.

## Comparison

| Feature | FDB (Boost.Asio) | ScyllaDB (Seastar) | TigerBeetle (direct) |
|---------|-------------------|--------------------|-----------------------|
| Network I/O | io_uring via Asio | epoll (!) | io_uring direct |
| Storage I/O | RocksDB io_uring | io_uring direct | io_uring direct |
| Registered buffers | No | Yes | No |
| IOPOLL | No | Yes | No |
| Zero-copy | No | No | No |
| Multishot | No | No | No |

FDB gets moderate benefit. The real win would require dropping Boost.Asio and going direct — but that's a massive rewrite for uncertain gains.
