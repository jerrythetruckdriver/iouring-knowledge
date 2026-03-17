# Rust io_uring Ecosystem

Rust has the richest io_uring ecosystem of any language. Also the most fragmented. Here's the lay of the land.

## The Core Problem

Rust's `AsyncRead`/`AsyncWrite` traits assume borrowed buffers. io_uring needs ownership — the kernel holds the buffer between submission and completion. This fundamental mismatch spawned an entire ecosystem of competing approaches.

## The Players

### tokio-uring
- **Model**: Tokio-compatible, single-threaded per runtime
- **Buffer**: Ownership-based `IoBuf`/`IoBufMut` traits
- **Status**: Active, but Tokio team treats it as second-class
- **Verdict**: Good for adding io_uring to existing Tokio apps. Limited by Tokio's poll-based core.

### monoio (ByteDance)
- **Model**: Thread-per-core, ownership-based I/O
- **Status**: Active, production use at ByteDance
- **Strengths**: Clean ownership model, good performance
- **Weaknesses**: Linux-focused, macOS via kqueue fallback

### glommio (Datadog → community)
- **Model**: Thread-per-core, direct io_uring integration
- **Status**: Active but slower development
- **Strengths**: Battle-tested at Datadog, good storage I/O patterns
- **Weaknesses**: Linux-only, complex API

### compio
- **Model**: Thread-per-core, cross-platform (IOCP/io_uring/polling)
- **Status**: Active, inspired by monoio
- **Strengths**: True cross-platform completion-based I/O. IOCP on Windows, io_uring on Linux, polling fallback elsewhere.
- **Weaknesses**: Younger project, smaller community

### ringbahn
- **Model**: Safe io_uring bindings, async/await compatible
- **Status**: Experimental/prototype. Last meaningful activity unclear.
- **Historical significance**: Demonstrated that safe, ergonomic io_uring bindings are *possible* in Rust. Influenced later designs.
- **Verdict**: Research project, not production-ready.

### nuclei
- **Status**: Appears abandoned or absorbed into other projects.

## Low-Level Bindings

### io-uring (Tokio org)
- Raw safe bindings to liburing. The foundation most higher-level crates build on.
- Active, well-maintained.

### uring-sys
- Raw FFI bindings. Less commonly used directly.

## The Landscape

```
                    Cross-platform?
                    │
        Yes ────────┤──────── No (Linux-only)
        │           │              │
     compio      tokio          monoio
                (epoll/kqueue)  glommio
                tokio-uring     ringbahn
```

## What's Missing

No crate has it all:
- **tokio-uring**: No multishot, no zero-copy rx, no registered buffers optimization
- **monoio**: No Windows, limited advanced io_uring features
- **compio**: Cross-platform but youngest
- **glommio**: No Windows/macOS, heavyweight

The Rust ecosystem keeps reinventing the wheel because nobody solved the ownership problem at the trait level. The `IoBuf` approach works but fragments the ecosystem into "tokio-compatible" vs "io_uring-native" camps.

## Recommendation

| Use case | Pick |
|----------|------|
| Existing Tokio app, add io_uring | tokio-uring |
| New Linux-only network service | monoio |
| New Linux-only storage service | glommio |
| Cross-platform completion I/O | compio |
| Raw control | io-uring crate directly |
