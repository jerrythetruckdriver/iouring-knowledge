# Elixir/BEAM VM and io_uring

## Current State

The BEAM VM (Erlang/OTP) does not use io_uring. As of OTP 27 (2024), the BEAM's I/O subsystem uses:
- **Networking**: Custom poll/epoll/kqueue integration via `erl_check_io`
- **File I/O**: Async thread pool (dirty I/O schedulers, `async_threads`)
- **NIF-based I/O**: Dirty NIF schedulers

No io_uring references exist in the OTP source tree beyond unrelated search hits.

## Why BEAM Doesn't Need io_uring (Much)

### The BEAM's I/O Model

BEAM processes are lightweight (2KB initial heap). The VM multiplexes millions of processes onto a fixed set of schedulers (typically 1 per CPU core). Each scheduler has its own poll set.

For networking, this works well:
- One `epoll_wait()` per scheduler, per reduction cycle
- Non-blocking sockets with message-passing completion
- The VM already batches I/O checks efficiently

The syscall overhead that io_uring eliminates is a small fraction of BEAM's per-message overhead (process scheduling, garbage collection, message copying).

### Where io_uring Could Help

**File I/O**: BEAM's dirty I/O schedulers use blocking read/write on dedicated threads. io_uring would:
- Eliminate thread pool overhead for file ops
- Enable linked write+fsync (WAL patterns)
- Allow O_DIRECT without tying up a scheduler thread

**High-throughput UDP**: For DNS servers, metrics collectors, or game servers on BEAM, io_uring's multishot recv and bundle ops could reduce per-packet overhead.

## NIF Approach

The most practical path: a NIF library that wraps io_uring for specific workloads.

```erlang
%% Hypothetical io_uring NIF for file I/O
{ok, Ring} = iouring_nif:create(256),
ok = iouring_nif:prep_write(Ring, Fd, Data, Offset),
ok = iouring_nif:submit(Ring),
{ok, Result} = iouring_nif:wait(Ring).
```

**NIF constraints**:
- Must not block a scheduler for > 1ms (use dirty NIF scheduler)
- Memory allocated in NIF must be tracked (resource objects)
- Ring should be bound to a dirty NIF scheduler thread (SINGLE_ISSUER)

**Dirty NIF schedulers** are actually well-suited: they're dedicated threads that can own a ring and run a submission/completion loop without affecting the main schedulers.

## Port Driver Approach

Alternative: implement as a port driver. The BEAM sends commands via port protocol, driver submits to io_uring, completions are sent back as Erlang messages.

More complex than NIF but better isolation — a crash in the port driver doesn't take down the VM.

## Existing Libraries

None mature. The Elixir/Erlang ecosystem is small enough that io_uring libraries haven't gained traction. Most Elixir developers target macOS (kqueue) as well, making Linux-only optimizations less attractive.

## Comparison with Go/Node

| Runtime | io_uring Status | Reason |
|---------|----------------|--------|
| Go | Pure-Go libraries (giouring) | Runtime impedance, but possible |
| Node.js (libuv) | File I/O only (13 ops) | Pragmatic scope |
| BEAM | Nothing | Scheduler model works, cross-platform priority |
| CPython | Nothing | GIL, cross-platform, asyncio uses epoll |
| JVM (Netty) | Full transport | JNI bridge, mature implementation |

## Would io_uring Help Erlang Performance?

For typical Erlang workloads (messaging systems, telecom, web backends): **marginal improvement at best**. The bottleneck is message passing and process scheduling, not syscall overhead.

For specialized workloads (database storage engines, file processing pipelines, high-throughput UDP): **meaningful improvement** via NIF, but you'd be fighting the BEAM's process model to get there.

If you need io_uring-level I/O performance, the BEAM is probably the wrong runtime. Use it for what it's great at (fault tolerance, soft real-time, distributed systems) and offload I/O-intensive work to a sidecar written in Rust/C/Zig.
