# io_uring + WireGuard

## How WireGuard Works

WireGuard is a kernel module that creates virtual network interfaces (`wg0`). It encrypts packets using ChaCha20-Poly1305, encapsulates them in UDP, and sends them to peers. The kernel module handles everything: handshakes, encryption, routing.

Key architecture detail: WireGuard operates at **Layer 3** (IP packets), not Layer 4 (sockets). There's no userspace data path for packet processing.

## io_uring Interaction: Minimal

### Reading/Writing the TUN Interface

WireGuard creates a network interface, not a character device. Applications don't read/write WireGuard directly — they use normal sockets that happen to route through `wg0`.

```
Application → io_uring SEND → socket → kernel routing → wg0 → WireGuard encrypt → UDP → NIC
```

io_uring helps the **application layer** (e.g., a proxy running over WireGuard), not WireGuard itself.

### UDP Socket for Custom WireGuard Implementations

The Go userspace implementation (`wireguard-go`) and `boringtun` (Cloudflare's Rust implementation) use UDP sockets + TUN devices. Here, io_uring could help:

```c
// wireguard-go style: read from TUN, encrypt, send UDP
// io_uring can async both sides:
io_uring_prep_read(sqe, tun_fd, pkt_buf, MTU, 0);   // read plaintext from TUN
// ... encrypt in userspace ...
io_uring_prep_send(sqe, udp_fd, encrypted, len, 0);  // send to peer
```

But: wireguard-go uses Go's runtime (no io_uring). boringtun uses epoll.

### Where io_uring Actually Helps

1. **Applications running over WireGuard** — your TCP/UDP server using io_uring benefits from async I/O regardless of whether traffic routes through wg0. WireGuard encryption happens transparently in the kernel.

2. **Userspace WireGuard implementations** — if someone wrote a WireGuard implementation using io_uring for the TUN + UDP socket I/O, you'd get:
   - Batched reads from TUN device
   - Batched UDP sends to peers
   - Linked read→encrypt→send chains (encrypt still userspace CPU)

3. **High-throughput VPN gateways** — server handling many WireGuard tunnels, forwarding traffic. io_uring for the forwarding path (recv from client → route → send to tunnel).

## Why WireGuard Doesn't Need io_uring Internally

The kernel WireGuard module processes packets in softirq/workqueue context. It:
- Receives UDP packets via kernel networking stack
- Decrypts in a dedicated `wg-crypt` workqueue (per-CPU threads)
- Injects decrypted packets back into the network stack

No syscalls in the fast path. No userspace involvement. io_uring's syscall batching is irrelevant here.

## Performance Reality

WireGuard's bottleneck is **crypto throughput**, not I/O. ChaCha20-Poly1305 on modern x86 with AVX2/AVX-512 does ~5-10 Gbps per core. The overhead is in encryption, not in moving packets between kernel and userspace.

For userspace implementations (wireguard-go, boringtun), the bottleneck is the TUN device read/write syscalls. io_uring would help here — but nobody's built it yet.

## What Would Be Interesting

A Rust/C userspace WireGuard implementation using:
- io_uring for TUN reads (multishot or batched)
- io_uring for UDP send/recv (multishot recv + bundle send)
- Registered buffers for packet buffers
- SQPOLL to eliminate syscalls entirely

This would outperform wireguard-go significantly on high-throughput gateways. The kernel module would still win for single-tunnel scenarios (no user↔kernel transitions at all).

## Bottom Line

io_uring and WireGuard are mostly orthogonal. WireGuard is a kernel-level packet processor. io_uring is a userspace↔kernel I/O interface. They meet at the edges: applications using sockets over WireGuard, or hypothetical userspace WireGuard implementations using io_uring for I/O.
