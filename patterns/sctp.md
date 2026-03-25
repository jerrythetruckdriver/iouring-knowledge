# SCTP (Stream Control Transmission Protocol)

## Status

SCTP works with io_uring via standard socket ops. No SCTP-specific opcodes exist.

## What Works

| Operation | Opcode | Notes |
|-----------|--------|-------|
| Socket creation | `SOCKET` | `AF_INET, SOCK_STREAM, IPPROTO_SCTP` or `SOCK_SEQPACKET` |
| Bind | `BIND` (6.14) | Standard bind |
| Listen | `LISTEN` (6.14) | Standard listen |
| Accept | `ACCEPT` | Multishot works, one-to-many (`SOCK_SEQPACKET`) accepted |
| Connect | `CONNECT` | Multi-homing: kernel handles path setup |
| Send | `SEND` / `SENDMSG` | `SENDMSG` needed for `sctp_sndrcvinfo` via cmsg |
| Recv | `RECV` / `RECVMSG` | `RECVMSG` needed for `SCTP_SNDRCV` cmsg (association ID, stream) |
| Poll | `POLL_ADD` | Association events, readability |
| Close | `CLOSE` | Triggers SHUTDOWN/SHUTDOWN-ACK |

## SCTP-Specific Concerns

**cmsg is mandatory.** SCTP multiplexes multiple streams over one socket. Stream ID, association ID, and PPID travel in ancillary data. You need `RECVMSG` (not `RECV`) to get them.

**Multishot RECVMSG with SCTP:**
- Works for `SOCK_SEQPACKET` (message-oriented)
- Buffer sizing: must accommodate largest SCTP message + cmsg overhead
- `io_uring_recvmsg_out` header parsing same as UDP

**One-to-many vs one-to-one:**
- `SOCK_STREAM` (one-to-one): behaves like TCP, all ops work identically
- `SOCK_SEQPACKET` (one-to-many): single socket handles all associations, `RECVMSG` critical for demuxing

## Socket Options via URING_CMD

SCTP socket options (`SCTP_NODELAY`, `SCTP_EVENTS`, `SCTP_MAXSEG`) work via `GETSOCKOPT`/`SETSOCKOPT` URING_CMD (6.11+).

## Pattern: SCTP Server

```
SOCKET(AF_INET, SOCK_SEQPACKET, IPPROTO_SCTP)
  → BIND → LISTEN
  → RECVMSG multishot (buffer group, reads all associations)
  → parse sctp_sndrcvinfo from cmsg for stream/assoc routing
  → SENDMSG with sctp_sndrcvinfo for reply targeting
```

No `ACCEPT` needed for one-to-many — associations are implicit.

## When io_uring Helps

SCTP is niche (telecom signaling, SIGTRAN). The real benefit is batching — a diameter/SCTP gateway handling thousands of associations on one socket can batch all recvmsg completions through a single ring instead of one epoll wake per message.

## What's Missing

- No SCTP-specific URING_CMD ops (sctp_peeloff, sctp_connectx)
- `sctp_peeloff()` to extract associations still requires syscall
- No multishot for `sctp_recvv()` (userspace sctp library function, not a syscall)
