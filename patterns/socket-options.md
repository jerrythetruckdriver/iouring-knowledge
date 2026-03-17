# Socket Options via io_uring

## URING_CMD Socket Operations

io_uring can get/set socket options without syscalls using `IORING_OP_URING_CMD` with socket-specific command opcodes.

```c
enum io_uring_socket_op {
    SOCKET_URING_OP_SIOCINQ,       // bytes available in recv buffer
    SOCKET_URING_OP_SIOCOUTQ,      // bytes pending in send buffer
    SOCKET_URING_OP_GETSOCKOPT,    // getsockopt() equivalent
    SOCKET_URING_OP_SETSOCKOPT,    // setsockopt() equivalent
    SOCKET_URING_OP_TX_TIMESTAMP,  // transmit hardware timestamps
    SOCKET_URING_OP_GETSOCKNAME,   // getsockname() equivalent
};
```

### getsockopt / setsockopt

```c
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
sqe->opcode = IORING_OP_URING_CMD;
sqe->fd = socket_fd;  // can be fixed file
sqe->cmd_op = SOCKET_URING_OP_SETSOCKOPT;
sqe->level = SOL_SOCKET;
sqe->optname = SO_RCVLOWAT;
sqe->optval = (uint64_t)&value;
sqe->optlen = sizeof(value);
```

This is async `setsockopt()`. No syscall, no context switch. Set `SO_RCVLOWAT` to batch small reads, adjust `SO_SNDBUF` dynamically, toggle `TCP_NODELAY` — all inline with your I/O submissions.

### SO_RCVLOWAT Integration

`SO_RCVLOWAT` via io_uring is particularly useful with multishot recv. Set a low watermark so the kernel only completes the recv when enough data is available:

```c
// Set RCVLOWAT to 4096 — don't wake me for less than 4KB
int val = 4096;
// submit SOCKET_URING_OP_SETSOCKOPT for SO_RCVLOWAT

// Then submit multishot recv — kernel waits for 4KB+ before completing
```

Reduces CQE churn on connections receiving many small packets. Instead of one CQE per 64-byte packet, wait until 4KB accumulates.

### TX Timestamps

```c
sqe->cmd_op = SOCKET_URING_OP_TX_TIMESTAMP;
```

Get hardware transmit timestamps asynchronously. CQE carries the timestamp:
- `IORING_CQE_F_TSTAMP_HW` flag indicates hardware timestamp
- Timestamp data in the CQE extra fields

Useful for high-frequency trading, network monitoring, PTP synchronization.

### SIOCINQ / SIOCOUTQ

Async equivalents of `ioctl(SIOCINQ)` and `ioctl(SIOCOUTQ)`. Query how many bytes are pending in the socket's receive or send buffer without a syscall.

Use case: adaptive batching — check SIOCINQ, if data is queued, submit a larger recv buffer.

## Why Not Just Use getsockopt(2)?

1. **Zero syscalls:** Socket option changes inline with I/O submissions
2. **Batching:** Set options on 1000 sockets in one `io_uring_enter()`
3. **Fixed files:** Works with registered file descriptors — no fget/fput
4. **Ordering:** Can link option changes with I/O ops — set NODELAY, then send, guaranteed order
