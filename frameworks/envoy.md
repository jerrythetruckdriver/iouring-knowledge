# Envoy Proxy

C++ L7 proxy. io_uring support is real but limited.

## Status: Experimental (since ~1.31, late 2024)

Envoy added io_uring as an optional I/O backend. It's behind a build flag and not the default path.

## Architecture

From `source/common/io/io_uring_impl.h`:

```cpp
class IoUringImpl : public IoUring, public ThreadLocal::ThreadLocalObject {
    struct io_uring ring_;
    os_fd_t event_fd_;
    std::list<InjectedCompletion> injected_completions_;
};
```

Key design decisions:
- **One ring per worker thread** (via `ThreadLocalObject`)
- **Eventfd bridge** to Envoy's existing event loop (libevent)
- **SQPOLL optional** via `use_submission_queue_polling` constructor param

## Supported Operations

```cpp
prepareAccept()
prepareConnect()
prepareReadv()
prepareWritev()
prepareClose()
prepareCancel()
prepareShutdown()
```

Basic socket ops only. No multishot accept, no multishot recv, no provided buffers, no zero-copy send. This is a conservative integration — wrapping existing operations in io_uring SQEs rather than redesigning around io_uring's advanced features.

## Integration Model

Envoy uses eventfd to bridge io_uring completions into its libevent loop:

```
io_uring CQE → eventfd write → libevent wakes → drain CQEs
```

This means Envoy doesn't get the full benefit of io_uring's zero-syscall completion path. Every batch of completions still triggers an eventfd write + libevent callback. Better than epoll's per-fd syscalls, but not io_uring-native.

## What's Missing

- **Multishot accept**: Would eliminate per-connection accept overhead
- **Multishot recv**: Would eliminate per-read SQE submission
- **Provided buffer rings**: Would eliminate buffer management overhead
- **Zero-copy send**: Would eliminate TX copies for large responses
- **Bundle ops**: Would reduce CQ pressure
- **Direct descriptors**: Would eliminate fd table lookups

In short: Envoy uses io_uring as a better epoll, not as a fundamentally different I/O model.

## Known Issues

From GitHub issues:
- **#36349**: liburing compilation fails on older kernels (RHEL 8, kernel 4.18). io_uring was added in 5.1 — Envoy needs graceful fallback.
- **#38660-38677**: Various io_uring-related bugs (2025), mostly around edge cases in the socket lifecycle.
- **#40838, #42562**: Ongoing work to stabilize and extend io_uring support.

## Build Configuration

io_uring is optional:
```
bazel build //source/exe:envoy-static --define=use_io_uring=true
```

Requires liburing headers and a kernel ≥5.1. Runtime detection via `isIoUringSupported()`.

## Why It's Slow to Mature

1. **Abstraction layer**: Envoy's Network::IoHandle abstraction was designed for epoll. io_uring's completion model doesn't map cleanly.
2. **libevent dependency**: The eventfd bridge adds overhead and complexity.
3. **Multiplatform**: Envoy runs on macOS (no io_uring). Platform abstraction limits how deep io_uring integration can go.
4. **Security**: Envoy runs in Kubernetes. Many clusters disable io_uring (seccomp default, RHEL kernel flag). Hard to adopt what you can't use.

## Trajectory

Envoy's io_uring integration will likely grow but stay conservative. The proxy's value is in L7 protocol handling, not raw I/O throughput. For Envoy, moving from epoll to basic io_uring gives moderate gains. The big wins (multishot, provided buffers, zero-copy) would require deeper architectural changes.

Compare with Seastar/ScyllaDB where io_uring is the I/O foundation, not a bolt-on.
