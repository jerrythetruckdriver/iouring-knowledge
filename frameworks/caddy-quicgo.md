# Caddy & quic-go

## Caddy

**Language:** Go
**I/O:** Standard Go net package (epoll via netpoller)
**io_uring:** No. Same Go runtime limitation as every Go server.

Caddy is a Go HTTP server. Go's `net` package uses epoll internally via the netpoller. There's no io_uring integration in Go's runtime, and Caddy doesn't use cgo.

## quic-go

**Language:** Go
**Protocol:** QUIC (HTTP/3 transport)
**I/O:** UDP sockets via Go's net package
**io_uring:** No.

quic-go is the dominant Go QUIC implementation. It uses standard Go UDP sockets with `ReadMsgUDP`/`WriteMsgUDP`. These map to `recvmsg`/`sendmsg` syscalls via Go's netpoller.

**Where io_uring would help quic-go:**

1. **UDP recv batching:** QUIC servers receive thousands of UDP datagrams per second. Multishot `RECVMSG` with provided buffers would eliminate per-packet syscalls. quic-go currently does `recvmmsg` via cgo for batch recv on Linux — io_uring would be better.

2. **GSO/GRO integration:** `sendmsg` with `UDP_SEGMENT` cmsg (Generic Segmentation Offload) batches multiple QUIC packets into one syscall. io_uring's `SENDMSG_ZC` could make this zero-copy too.

3. **NAPI busy poll:** High-throughput QUIC benefits from polling the UDP socket's NIC queue directly.

**Why it won't happen:**
- Go runtime manages socket I/O scheduling. Can't bypass it without unsafe cgo.
- quic-go already uses `recvmmsg`/`sendmmsg` via cgo for Linux fast path — diminishing returns.
- Cross-platform is important (quic-go runs on macOS, Windows, FreeBSD).

## The Go + io_uring Gap

| What quic-go does | What io_uring could do | Delta |
|-------------------|----------------------|-------|
| recvmmsg (batch recv, cgo) | Multishot RECVMSG + provided buffers | Eliminate cgo overhead, zero-copy |
| sendmmsg (batch send, cgo) | Bundle SEND + SEND_ZC | Zero-copy, kernel-managed batching |
| epoll + read loop | NAPI busy poll | Skip interrupt, poll NIC directly |
| Per-packet GRO parse | Hardware GRO + zcrx | Zero-copy to application buffer |

The theoretical gains are 2-5x throughput for UDP-heavy QUIC workloads. But Go doesn't have the plumbing to get there.

## Who Gets QUIC + io_uring?

- **C/C++ QUIC implementations:** MsQuic (Microsoft), ngtcp2, picoquic — could adopt io_uring
- **Rust:** Quinn (via tokio-uring or io-uring crate) — possible but no one's done it
- **Zig:** std.Io.Evented + custom QUIC — the cleanest path

Nobody has shipped a production QUIC stack with io_uring yet. The protocol's complexity (encryption, congestion control, stream multiplexing) dwarfs the I/O layer gains.
