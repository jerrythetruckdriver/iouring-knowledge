# tokio-uring

io_uring integration for the Tokio async runtime ecosystem.

## What It Is

A Tokio-compatible runtime that uses io_uring for I/O instead of epoll. Not a replacement for Tokio — a complement. Runs alongside Tokio's work-stealing scheduler.

## Architecture

- Uses Tokio for task scheduling
- Replaces epoll-based I/O with io_uring
- One ring per worker thread
- Based on the `io-uring` crate for low-level bindings

## Ownership Model

Like Monoio, uses buffer ownership transfer:

```rust
let buf = vec![0u8; 4096];
let (res, buf) = file.read_at(buf, 0).await;
```

Buffer is consumed on submission, returned on completion. No use-after-submit possible.

## Limitations

- Not a full runtime replacement — Tokio's multi-threaded scheduler still manages tasks
- Work-stealing doesn't align perfectly with io_uring's `SINGLE_ISSUER`
- Feature coverage lags behind liburing
- Less battle-tested than raw Tokio

## When to Use

- Existing Tokio application that needs io_uring for specific hot paths
- File I/O in async Tokio context (Tokio's `fs` uses thread pool; tokio-uring uses io_uring)
- Gradual migration from epoll to io_uring

## Source

- Repository: https://github.com/tokio-rs/tokio-uring
- License: MIT
