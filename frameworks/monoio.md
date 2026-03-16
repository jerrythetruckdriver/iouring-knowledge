# Monoio

Thread-per-core async runtime from ByteDance. Production-deployed at massive scale.

## Architecture

Similar to Glommio — one io_uring per core, no work stealing. But with some differences:

- Tighter integration with Rust's async ecosystem
- `GAT`-based (Generic Associated Types) I/O traits for zero-cost buffer ownership
- epoll fallback for older kernels
- macOS support via kqueue

## io_uring Features Used

- `DEFER_TASKRUN` + `SINGLE_ISSUER` (preferred path)
- Provided buffer rings
- Multishot operations
- Fixed files and buffers
- `COOP_TASKRUN`

## Key Innovation: Ownership-Based I/O

Monoio's `AsyncReadRent` / `AsyncWriteRent` traits take buffer ownership:

```rust
// Buffer moves into the read operation
let (result, buf) = file.read(buf).await;
// Buffer comes back after completion
```

This solves the lifetime problem with completion-based I/O. The buffer can't be accessed while the kernel holds it. Compile-time safety.

## Production at ByteDance

Running in ByteDance's infrastructure serving millions of requests/sec. The thread-per-core model + io_uring combination handles their network proxy and storage workloads.

## Source

- Repository: https://github.com/bytedance/monoio
- License: Apache-2.0/MIT
