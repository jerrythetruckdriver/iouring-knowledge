# HTTP/3 and QUIC Server Patterns

## TL;DR

QUIC is UDP-based. io_uring's multishot UDP recv + bundle send + NAPI busy poll makes it the ideal QUIC packet engine. Nobody's shipping it yet.

## Why QUIC + io_uring is a Natural Fit

HTTP/3 runs over QUIC, which runs over UDP. This changes the game:

- **No kernel TCP state** — QUIC handles congestion, retransmission, stream mux in userspace
- **High packet rate** — Each QUIC connection generates many small UDP datagrams
- **Timestamp-sensitive** — ACK timing affects congestion control

io_uring's UDP capabilities map perfectly:

| QUIC need | io_uring feature |
|-----------|------------------|
| Receive many datagrams | Multishot RECVMSG (one SQE, continuous CQEs) |
| Send many datagrams | Bundle SEND (one SQE, multiple buffers) |
| Low-latency recv | NAPI busy poll (no interrupt, DMA → userspace) |
| TX timestamps | SOCKET_URING_OP_TX_TIMESTAMP via URING_CMD |
| Socket options | GETSOCKOPT/SETSOCKOPT via URING_CMD |
| GRO/GSO | recvmsg/sendmsg with cmsg for UDP_GRO/UDP_SEGMENT |

## The Packet Engine Pattern

```
// QUIC packet engine with io_uring
// One ring per worker thread (thread-per-core)

Setup:
  SINGLE_ISSUER | COOP_TASKRUN | NO_SQARRAY
  Register provided buffer ring (2KB buffers for QUIC MTU)
  REGISTER_NAPI for busy polling

Receive path:
  Submit one multishot RECVMSG on UDP socket
  → Continuous CQEs with io_uring_recvmsg_out header
  → Parse QUIC packet from buffer
  → cmsg gives SO_TIMESTAMPNS for ACK timing
  → IOU_PBUF_RING_INC for incremental consumption

Send path:
  Batch QUIC packets into provided buffer ring
  Submit bundle SENDMSG (one SQE, many datagrams)
  → Or SENDMSG_ZC for zero-copy if packets are large
  → UDP_SEGMENT cmsg for GSO (one syscall, kernel segments)

Timer path:
  TIMEOUT_MULTISHOT for connection idle timers
  Link timeouts for retransmission deadlines
```

## Buffer Sizing

QUIC datagrams are small (typically 1200-1350 bytes, max ~1472 for non-jumbo Ethernet).

- **Receive buffers:** 2KB each (1350 payload + cmsg overhead)
- **Buffer ring entries:** 4096+ for high connection counts
- **GRO coalesced:** Up to 64KB per recvmsg when UDP_GRO enabled

## What's Missing

No major QUIC implementation uses io_uring for its packet engine:

| Implementation | Language | I/O model |
|---------------|----------|-----------|
| quiche (Cloudflare) | Rust | mio (epoll) |
| quinn | Rust | tokio (epoll) |
| ngtcp2 | C | Poll/epoll |
| msquic | C | epoll/kqueue/IOCP |
| s2n-quic (AWS) | Rust | tokio (epoll) |
| quic-go | Go | net.PacketConn (epoll) |

**Why not adopted:**
1. Cross-platform requirement (QUIC runs on macOS/Windows too)
2. QUIC libraries expose a `sendmsg`/`recvmsg` interface — callers can use io_uring underneath
3. recvmmsg()/sendmmsg() already batch reasonably well for most loads
4. UDP GRO/GSO captures most of the kernel-side wins

## The GSO/GRO Question

Linux's UDP_SEGMENT (GSO) and UDP_GRO already reduce per-packet overhead dramatically:
- GSO: one `sendmsg()` with 64KB buffer → kernel segments into MTU-sized packets
- GRO: kernel coalesces packets → one `recvmsg()` returns 64KB

With GSO/GRO, the syscall reduction from io_uring is less dramatic. The win shifts to:
- Eliminating remaining syscalls (one multishot recv vs repeated recvmsg)
- NAPI busy poll for latency-sensitive QUIC (trading, gaming)
- TX timestamps without separate cmsg polling

## When io_uring QUIC Matters

- **>100K concurrent QUIC connections** — Syscall batching compounds
- **Low-latency QUIC** (trading, gaming) — NAPI + SQPOLL eliminates scheduling jitter
- **High-throughput QUIC proxy** — Zero-copy forward between sockets
- **Custom QUIC on Linux-only deployment** — No portability constraint

For most QUIC servers, epoll + GSO/GRO + recvmmsg is good enough.
