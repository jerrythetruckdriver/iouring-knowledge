# MPTCP (Multipath TCP) and io_uring

## TL;DR

MPTCP works transparently with io_uring. It's a kernel transport layer protocol — io_uring submits socket ops, kernel handles the multipath logic. No special opcodes needed.

## How It Works

MPTCP (RFC 8684) creates multiple TCP subflows over different network paths (e.g., WiFi + cellular). The kernel's MPTCP implementation handles:

- Subflow establishment and teardown
- Packet scheduling across paths
- Sequence number mapping (MPTCP → TCP per subflow)
- Congestion control per subflow

From userspace (and io_uring's perspective), an MPTCP socket behaves like a regular TCP socket.

## Creating MPTCP Sockets with io_uring

```c
// IORING_OP_SOCKET with IPPROTO_MPTCP
struct io_uring_sqe *sqe = io_uring_get_sqe(ring);
io_uring_prep_socket(sqe, AF_INET, SOCK_STREAM, IPPROTO_MPTCP, 0);

// Everything else is identical to TCP:
// BIND → LISTEN → ACCEPT → RECV → SEND → CLOSE
```

`IPPROTO_MPTCP` (262) is the only difference from regular TCP. All io_uring ops work:

| Operation | Works with MPTCP? |
|-----------|-------------------|
| SOCKET | Yes (IPPROTO_MPTCP) |
| BIND/LISTEN/ACCEPT | Yes |
| RECV/SEND (multishot, bundle, ZC) | Yes |
| CONNECT | Yes |
| SETSOCKOPT via URING_CMD | Yes |
| SHUTDOWN/CLOSE | Yes |

## MPTCP-Specific Socket Options

Configurable via `SOCKET_URING_OP_SETSOCKOPT`:

- `MPTCP_INFO` — Connection-level info (similar to TCP_INFO)
- `MPTCP_TCPINFO` — Per-subflow TCP info
- `MPTCP_SUBFLOW_ADDRS` — List subflow addresses
- `MPTCP_FULL_INFO` — Combined info

These are `SOL_MPTCP` level options. Work with io_uring's URING_CMD socket ops.

## Performance Considerations

- MPTCP adds ~2-5% CPU overhead vs TCP (kernel path scheduling, sequence mapping)
- io_uring's syscall batching helps amortize this overhead
- Multishot recv works — CQEs arrive from whichever subflow delivers data first
- Zero-copy send works but each subflow may need different segmentation

## When to Combine

MPTCP + io_uring makes sense for:
- Mobile servers (clients switching WiFi ↔ cellular)
- High-availability services (redundant network paths)
- Bandwidth aggregation (bonding paths)

The io_uring angle: if you're already using io_uring for your TCP server, switching to MPTCP is literally changing one constant (`IPPROTO_TCP` → `IPPROTO_MPTCP`).
