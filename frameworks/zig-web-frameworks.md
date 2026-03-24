# Zig Web Frameworks and io_uring

## Framework Survey

### Zap (zigzap/zap)

- **I/O backend**: facil.io (C library wrapper)
- **Event model**: epoll/kqueue via facil.io's evented networking
- **io_uring**: No. facil.io doesn't use io_uring
- **Status**: Production-used, Zig 0.15.1, blazingly fast benchmarks

Zap wraps facil.io — a battle-tested C web framework. This means io_uring adoption depends on facil.io upstream, which hasn't added it.

### http.zig (karlseguin/http.zig)

- **I/O backend**: Pure Zig, uses `std.posix` (epoll/kqueue)
- **Thread model**: Thread pool with blocking accept + per-connection handling
- **io_uring**: No. Uses `posix.accept()`, standard blocking I/O per connection
- **Status**: Active, Zig 0.15.1, ~140K req/s on M2

Pure Zig implementation, but follows the traditional accept-per-connection-thread-pool model. No async I/O.

### JetZig / Tokamak

Built on top of http.zig. Same I/O characteristics.

## The Zig std.Io.Evented Connection

Zig's standard library landed io_uring as a swappable I/O backend (Feb 2026, std.Io.Evented). This is the path for Zig web frameworks to get io_uring:

1. Framework builds on `std.Io` interface
2. Runtime selects io_uring backend on Linux
3. Green threads/fibers make async transparent

**No existing Zig web framework uses std.Io.Evented yet.** The API is new (Feb 2026) and frameworks haven't migrated.

## Why No Adoption Yet

1. **std.Io.Evented is brand new** — shipped Feb 2026, still stabilizing
2. **facil.io wrapper (Zap)** — can't use Zig std.Io, locked to C library's event model
3. **http.zig** — pure Zig but blocking thread-pool model, would need rewrite
4. **Zig doesn't have a tokio-equivalent** — no dominant async runtime ecosystem yet

## What Would Change Things

A Zig web framework built ground-up on `std.Io` with io_uring backend would be interesting. The fiber model means application code doesn't change — the runtime handles async transparently.

TigerBeetle already shows Zig + io_uring works exceptionally well at the systems level. A web framework following that pattern (direct ring manipulation, thread-per-core) could be competitive.

## Comparison

| Framework | Language | io_uring | Why/Why Not |
|-----------|----------|----------|-------------|
| Zap | Zig (facil.io) | ❌ | C library dependency |
| http.zig | Zig | ❌ | Blocking thread pool |
| JetZig | Zig (http.zig) | ❌ | Inherits http.zig |
| TigerBeetle | Zig | ✅ | Direct ring, not a web framework |
| std.Io.Evented | Zig stdlib | ✅ | Available, no framework uses it yet |

## Prediction

Within 12-18 months, someone will build a Zig HTTP server on std.Io.Evented with io_uring. The fiber model makes it natural. Whether it displaces Zap/http.zig depends on benchmarks and API ergonomics.
