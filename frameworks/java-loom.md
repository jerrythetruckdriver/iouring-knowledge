# Java Project Loom and io_uring

## TL;DR

Project Loom (virtual threads, JDK 21+) uses epoll for async I/O under the hood. No io_uring integration in the JVM. Netty's io_uring transport is the only Java io_uring path.

## Project Loom Architecture

Virtual threads (JEP 444, GA in JDK 21) implement cooperative scheduling:

1. Virtual thread calls `Socket.read()` (blocking API)
2. JVM detects socket not ready
3. Registers socket with internal poller (epoll on Linux)
4. Unmounts virtual thread from carrier thread (yields continuation)
5. Socket becomes ready → poller wakes → remounts virtual thread
6. `read()` returns to user code

**The poller is epoll.** `sun.nio.ch.EPoll` on Linux. No io_uring.

## Why No io_uring in the JVM

1. **Cross-platform requirement** — JVM runs on Linux, macOS, Windows, AIX, z/OS. io_uring is Linux-only. The JDK team won't add a Linux-specific I/O backend for the platform-agnostic `java.net` API.

2. **Virtual threads don't do I/O** — Virtual threads park/unpark around blocking calls. The actual I/O is still a regular syscall (`read`/`write`/`sendmsg`). The poller just tells the scheduler when to unpark. Replacing epoll with io_uring for polling gives minimal benefit.

3. **No completion-based I/O in java.net** — Java's `InputStream`/`OutputStream` are synchronous APIs. Even `java.nio.channels` with `Selector` is readiness-based, not completion-based. Changing to completion-based I/O would be a fundamental API redesign.

4. **Diminishing returns** — Virtual threads eliminate the need for async frameworks (Netty, Vert.x) for most applications. The remaining bottleneck is rarely syscall overhead.

## Netty: The io_uring Path

Netty's io_uring transport (mainlined in 4.2) is the only production Java io_uring usage:

- JNI bridge to liburing
- One ring per EventLoop
- True completion-based I/O (not poll-wait-syscall)
- See [netty.md](netty.md) for details

**Netty + Loom?** Technically possible but pointless. Netty already has its own async model. Using virtual threads with Netty adds overhead without benefit.

## Where Java + io_uring Makes Sense

| Use case | Path |
|----------|------|
| High-throughput network server | Netty io_uring transport |
| Database client (many connections) | Netty-based driver + io_uring |
| File I/O (database engine) | JNI + liburing directly |
| General web application | Virtual threads (epoll) — io_uring unnecessary |

## The Real Question

For most Java applications, virtual threads + epoll is fast enough. The overhead of JNI calls to io_uring often exceeds the syscall savings for typical workloads.

io_uring matters for Java when:
- You're writing a database engine (RocksDB-Java wrapper, etc.)
- You need >100K concurrent connections on Netty
- You're doing heavy file I/O with fsync (WAL paths)

For everything else, `Thread.ofVirtual().start(runnable)` and forget about it.
