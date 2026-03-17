# Send MSG_RING Without a Ring (IORING_REGISTER_SEND_MSG_RING)

Opcode 31. Send a message to another io_uring ring from a context that doesn't have a ring.

## The Problem

`IORING_OP_MSG_RING` lets you post a CQE to another ring — but you need a ring to submit the SQE. What if a thread that doesn't own a ring needs to signal one?

Before this: use eventfd, which the target ring polls. Extra fd, extra syscall, extra overhead.

## The Solution

```c
io_uring_register(target_ring_fd, IORING_REGISTER_SEND_MSG_RING,
                  &msg, 1);
```

This is a `register` operation, not a submission. Any thread can call `io_uring_register()` — it doesn't need its own ring. The target ring receives a CQE as if `IORING_OP_MSG_RING` was used.

## Use Cases

### Worker Thread → Event Loop Signaling

Classic producer-consumer:
```
Worker thread (no ring): IORING_REGISTER_SEND_MSG_RING → event loop ring
Event loop: receives CQE with result data, processes it
```

No eventfd. No pipe. No socket pair. Just a CQE appearing on the target ring.

### Cross-Process Notification (with registered ring fd)

If the target ring fd is shared (via SCM_RIGHTS or fixed fd install), another process can signal it without having its own ring.

## Compared to IORING_OP_MSG_RING

| | OP_MSG_RING | REGISTER_SEND_MSG_RING |
|---|---|---|
| Needs source ring | Yes | No |
| Async | Yes (CQE on source) | No (synchronous register call) |
| Batching | Yes (with other SQEs) | No |
| Use case | Ring-to-ring comms | External-to-ring signaling |

## Kernel Version

- **6.10**: Added
