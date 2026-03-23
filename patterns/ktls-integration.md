# io_uring + kTLS Integration

## What kTLS Is

Kernel TLS (kTLS) moves TLS record-layer encryption into the kernel. After userspace completes the handshake (OpenSSL, GnuTLS, etc.), it installs crypto state via `setsockopt(SOL_TLS, TLS_TX/TLS_RX)`. From that point, `send()`/`recv()` are transparently encrypted/decrypted.

Three modes:
- **TLS_SW** — CPU handles crypto in kernel
- **TLS_HW** — NIC handles per-packet crypto (Mellanox, Intel NICs)
- **TLS_HW_RECORD** — NIC replaces TCP stack entirely (not production-viable)

## io_uring + kTLS: It Just Works

Once kTLS is installed on a socket, io_uring operations on that socket automatically get TLS encryption/decryption. No special io_uring setup required.

```
// After TLS handshake + setsockopt(SOL_TLS, TLS_TX, ...)
// io_uring SEND/RECV on this fd → encrypted/decrypted transparently
io_uring_prep_send(sqe, tls_fd, plaintext, len, 0);
io_uring_prep_recv(sqe, tls_fd, buf, buflen, 0);
```

Works with:
- `IORING_OP_SEND` / `IORING_OP_RECV` — basic TLS I/O
- `IORING_OP_SENDMSG` / `IORING_OP_RECVMSG` — for control messages (record type via cmsg)
- `IORING_OP_SPLICE` — sendfile-style TLS transmission
- Multishot recv — continuous TLS decryption

## What Doesn't Work (or Gets Complicated)

### Zero-Copy Send (SEND_ZC)
kTLS needs to encrypt data before transmission. With `TLS_TX_ZEROCOPY_RO` + HW offload, the NIC encrypts in-flight, so zero-copy works — but only if:
1. Hardware offload is active (TLS_HW mode)
2. Data is read-only between submit and completion
3. NIC supports it (Mellanox ConnectX-6+, Intel E810)

Without HW offload, the kernel copies and encrypts, so SEND_ZC buys you nothing.

### Zero-Copy Receive (zcrx)
zcrx operates below the TLS layer. Data arrives encrypted in zcrx buffers, then kTLS decrypts into userspace buffers on recv. The DMA-to-userspace path still helps (one fewer copy), but you don't get plaintext directly in the zero-copy buffer.

### Provided Buffers
Work fine. kTLS decrypts into whatever buffer you provide. The kernel handles TLS record boundaries — a single recv may return less than a full buffer if it hits a record boundary.

### Bundle Recv
Works, but each buffer gets at most one TLS record's worth of data. Bundle efficiency depends on record sizes (typically ≤16KB).

## kTLS + io_uring Performance Pattern

```c
// 1. Standard TCP connection via io_uring
io_uring_prep_socket(sqe, AF_INET, SOCK_STREAM, 0, 0);
io_uring_prep_connect(sqe, fd, addr, addrlen);

// 2. TLS handshake in userspace (OpenSSL/etc)
// Can't do this in io_uring — handshake is complex stateful protocol

// 3. Install kTLS via setsockopt (one syscall, not performance-critical)
setsockopt(fd, SOL_TCP, TCP_ULP, "tls", 4);
setsockopt(fd, SOL_TLS, TLS_TX, &crypto_info, sizeof(crypto_info));
setsockopt(fd, SOL_TLS, TLS_RX, &crypto_info_rx, sizeof(crypto_info_rx));

// 4. Data path — fully async via io_uring
// Multishot recv for continuous decrypted reads
io_uring_prep_recv_multishot(sqe, fd, 0, 0);
sqe->flags |= IOSQE_BUFFER_SELECT;
sqe->buf_group = group_id;
```

## TLS 1.3 Key Updates

kTLS handles TLS 1.3 KeyUpdate messages. When a KeyUpdate is received:
1. Kernel pauses decryption
2. `recv()` / io_uring RECV returns `-EKEYEXPIRED`
3. Userspace installs new key via `setsockopt(TLS_RX, ...)`
4. Reads resume

For io_uring multishot recv: the multishot terminates on EKEYEXPIRED. Resubmit after key update.

## Supported Ciphers

- AES-GCM-128 (most common, best HW offload support)
- AES-GCM-256
- AES-CCM-128
- ChaCha20-Poly1305 (good for non-AES-NI CPUs)
- SM4 GCM/CCM (Chinese national standard)

## Hardware Offload + io_uring

The real win: kTLS HW offload + io_uring eliminates both syscall overhead AND crypto CPU load.

```
Without:  userspace → syscall → kernel encrypt → NIC TX
With:     userspace → io_uring SQE → NIC encrypts + TX (zero CPU crypto)
```

NICs with TLS offload: Mellanox ConnectX-5+, Intel E810, Broadcom P2100.

## When to Use

| Scenario | Recommendation |
|----------|---------------|
| HTTPS server, HW offload NIC | kTLS + io_uring + SEND_ZC = optimal |
| HTTPS server, no HW offload | kTLS + io_uring still good (saves context switches) |
| Proxy/load balancer | kTLS for backend connections, full TLS for frontend |
| Internal service mesh | Consider mTLS with kTLS for performance |
| High-connection-count | kTLS + multishot recv + provided buffers |

## Limitations

- **Handshake stays in userspace** — io_uring can't do TLS negotiation
- **setsockopt not async** — key installation requires syscall (one-time cost per connection)
- **Record boundaries visible** — recv may return partial data at record boundaries
- **No QUIC** — kTLS is TCP-only; QUIC/UDP-TLS must be userspace
- **Key rotation** — interrupts multishot recv flow
