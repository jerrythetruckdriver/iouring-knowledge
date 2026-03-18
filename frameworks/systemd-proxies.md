# systemd and Proxy Servers

## systemd

**Status: No io_uring usage.**

systemd's event loop (`sd-event`) is built on epoll + timerfd + signalfd + eventfd. io_uring only appears in systemd's codebase in seccomp syscall filter lists — it knows io_uring syscalls exist so it can block or allow them in sandboxed services.

### Why Not

1. **Portability.** systemd targets every Linux kernel ≥ 4.15 (as of 2026). io_uring requires 5.1 minimum, and useful features require 6.x. They can't drop support for older kernels.
2. **sd-event is not hot path.** systemd's event loop manages service lifecycle, dbus messages, and journal writes — not high-throughput I/O. The syscall overhead that io_uring eliminates isn't systemd's bottleneck.
3. **Security posture.** systemd heavily uses seccomp and namespacing. io_uring's CVE history and attack surface expansion runs counter to systemd's security-first approach. Many systemd service sandboxes explicitly block io_uring syscalls via `SystemCallFilter=`.

### Journal Writes

The one area where io_uring might help: `systemd-journald` writes structured log entries under high throughput. But journald uses `writev()` and `fdatasync()`, and the bottleneck is usually disk, not syscall overhead.

### Verdict

systemd won't adopt io_uring until it's been stable and ubiquitous for years. Given that RHEL 9 has io_uring disabled by default, "ubiquitous" isn't there yet.

## HAProxy

**Status: No io_uring usage.**

HAProxy's event loop is deeply integrated with epoll (Linux), kqueue (BSD), and poll (fallback). The codebase has zero io_uring references.

### Why Not

1. **Mature poll engine.** HAProxy's epoll integration is battle-tested at massive scale. The threading model (one event loop per thread, shared fd tables) is optimized for epoll's semantics.
2. **Cross-platform.** HAProxy runs on Linux, FreeBSD, macOS. io_uring is Linux-only. Adding a Linux-only fast path means maintaining two codepaths.
3. **Diminishing returns.** HAProxy is already extremely efficient. The marginal gain from io_uring over epoll for a proxy workload (mostly `recv` → `send` with small buffers) is minimal compared to the engineering cost.

## Traefik

**Status: No io_uring usage.**

Written in Go. Go's runtime has its own epoll-based netpoller baked into the scheduler. There's no practical way to integrate io_uring without bypassing Go's I/O model entirely.

Same fundamental problem as Caddy: Go's goroutine scheduler assumes blocking I/O that's secretly async via runtime netpoller. io_uring's completion-based model doesn't map to this.

## Pattern: Why L7 Proxies Don't Use io_uring

L7 proxies share characteristics that make io_uring adoption unlikely:

1. **Recv/send dominated.** Proxy workloads are 90% recv+send. The syscall overhead difference between epoll+recv+send vs io_uring is real but small — not enough to justify a rewrite.
2. **Connection-per-fd scale.** Proxies handle many connections with small payloads. io_uring shines on batch I/O with large buffers, not thousands of small recv calls.
3. **Feature maturity.** Proxies need bulletproof TLS, HTTP parsing, health checks, retries. The I/O layer is a small part of the complexity.
4. **Operator trust.** Proxy operators are conservative. New kernel APIs with CVE histories don't get adopted quickly in infrastructure that routes production traffic.

### Exception: Envoy

Envoy has an experimental io_uring backend (see [envoy.md](envoy.md)) but it's incomplete — no multishot, no provided buffers, no zero-copy. It exists as a proof of concept, not a production option.

### Future

The proxy that benefits most from io_uring will be one built from scratch in a language with first-class io_uring support (Rust/Zig), not an existing epoll-based proxy with io_uring bolted on.
