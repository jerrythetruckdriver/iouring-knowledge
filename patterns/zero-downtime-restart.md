# Zero-Downtime Restart

Hot restart without dropping connections. io_uring provides the building blocks but doesn't solve it alone.

## The Strategy

1. Old process stops accepting
2. Pass listening fd(s) to new process
3. Old process drains in-flight requests
4. Old process exits

## Fd Passing Methods

### Unix Socket + SCM_RIGHTS

Traditional. Works with io_uring — use `SENDMSG`/`RECVMSG` with SCM_RIGHTS cmsg for fd passing between old and new process.

```
Old process:
  SENDMSG(unix_sock, fd_array via SCM_RIGHTS)

New process:
  RECVMSG(unix_sock) → extract fds from cmsg
  → re-register fds as fixed files
  → start multishot ACCEPT on received listening fds
```

### MSG_RING Fd Passing (6.0+)

If both processes share a ring reference (e.g., via fork or fd passing):

```
Old process:
  MSG_RING(new_ring_fd, IORING_MSG_SEND_FD, src_fd, dst_fd)
```

This installs a file descriptor directly into the target ring's fixed file table. Faster than SCM_RIGHTS — no sendmsg overhead.

### pidfd_getfd (5.6+)

New process can steal fds from old process via `pidfd_getfd()`. No cooperation needed from old process. Useful for crash recovery.

## Graceful Drain Pattern

```
Old process:
1. Stop submitting new ACCEPT ops (or cancel multishot ACCEPT)
2. For each in-flight connection:
   - Let current request complete
   - Submit SHUTDOWN(SHUT_WR) when response sent
   - Wait for peer close (RECV returns 0)
   - CLOSE
3. Cancel all remaining ops (ASYNC_CANCEL with CANCEL_ANY | CANCEL_ALL)
4. Drain CQ until empty
5. Close ring, exit
```

**Critical:** Don't just close the ring. In-flight requests may reference memory (registered buffers, provided buffer rings). Cancel and drain first.

## Ring State Transfer

io_uring rings themselves are **not transferable.** Each ring is bound to the creating process/thread. You can transfer:

- File descriptors (listening sockets, client connections)
- Application state (serialized via shared memory or pipe)

You **cannot** transfer:
- The ring itself
- Registered buffers
- In-flight SQEs
- SQPOLL thread

The new process must create a fresh ring and re-register everything.

## Complete Pattern

```
┌─────────────┐     unix socket      ┌─────────────┐
│ Old Process  │ ──── SCM_RIGHTS ───→ │ New Process  │
│              │    (listening fds)    │              │
│ 1. stop accept                      │ 2. new ring  │
│ 3. drain     │                      │ 4. register  │
│ 4. exit      │                      │ 5. accept    │
└─────────────┘                       └─────────────┘
```

## SO_REUSEPORT Alternative

With SO_REUSEPORT, zero-downtime is simpler:

1. New process starts, creates its own SO_REUSEPORT sockets
2. Kernel starts routing new connections to new process
3. Old process stops accepting, drains existing connections
4. Old process exits

No fd passing needed. Each process is independent. This is how Envoy, Nginx, and most modern servers do it.

**Downside:** Brief period where both old and new process accept. Connections started on old process stay there until drained.

## Gotchas

- **Direct descriptors don't survive ring teardown.** Export with `FIXED_FD_INSTALL` before closing old ring.
- **SQPOLL thread is per-ring.** New ring, new SQPOLL thread. No continuity.
- **Registered buffers are per-ring.** Re-register in new ring.
- **In-flight multishot ops terminate on cancel.** Expect `-ECANCELED` CQEs during drain.
