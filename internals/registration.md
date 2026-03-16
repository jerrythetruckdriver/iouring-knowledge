# Registration Operations

`io_uring_register()` operations for pre-registering resources. Eliminates per-operation overhead of fd lookups and buffer mapping.

## Buffer Registration

### `IORING_REGISTER_BUFFERS` / `IORING_UNREGISTER_BUFFERS`
Pin user buffers in kernel memory. Use with `IORING_OP_READ_FIXED` / `IORING_OP_WRITE_FIXED`. Avoids `get_user_pages()` on every I/O — huge for high-throughput storage.

### `IORING_REGISTER_BUFFERS2` / `IORING_REGISTER_BUFFERS_UPDATE`
Tagged buffer registration. Allows sparse registration and incremental updates.

### `IORING_REGISTER_CLONE_BUFFERS` (6.12)
Clone registered buffers from one ring to another. Avoids re-pinning the same pages.

## File Registration

### `IORING_REGISTER_FILES` / `IORING_UNREGISTER_FILES`
Pre-register file descriptors. Reference via index instead of fd. Skips `fget()`/`fput()` per operation. Use `IOSQE_FIXED_FILE` flag.

### `IORING_REGISTER_FILES2` / `IORING_REGISTER_FILES_UPDATE2`
Tagged file registration with sparse support. `IORING_RSRC_REGISTER_SPARSE` creates a fully sparse table (all slots -1).

### `IORING_REGISTER_FILE_ALLOC_RANGE` (6.3)
Reserve a range of file slots for auto-allocation. Opcodes like `accept` and `openat` with `IORING_FILE_INDEX_ALLOC` pick from this range.

## Provided Buffer Rings

### `IORING_REGISTER_PBUF_RING` / `IORING_UNREGISTER_PBUF_RING` (5.19)
Register a ring-based provided buffer pool. Superior to `PROVIDE_BUFFERS` opcode — no SQE needed to replenish.

Flags:
- `IOU_PBUF_RING_MMAP` — kernel allocates ring memory
- `IOU_PBUF_RING_INC` — incremental consumption (6.10+). Buffers partially consumed, tracked by kernel and app.

### `IORING_REGISTER_PBUF_STATUS` (6.4)
Query buffer group head position. Debugging aid.

## Ring Management

### `IORING_REGISTER_RING_FDS` / `IORING_UNREGISTER_RING_FDS` (5.18)
Register the ring fd itself. Use with `IORING_ENTER_REGISTERED_RING` to avoid fd table lookup on `io_uring_enter()`.

### `IORING_REGISTER_ENABLE_RINGS` (5.10)
Enable a ring created with `IORING_SETUP_R_DISABLED`.

### `IORING_REGISTER_RESIZE_RINGS` (6.13)
Resize CQ ring without recreating the ring. Finally.

## Worker Configuration

### `IORING_REGISTER_IOWQ_AFF` / `IORING_UNREGISTER_IOWQ_AFF` (5.14)
Set CPU affinity for io-wq worker threads.

### `IORING_REGISTER_IOWQ_MAX_WORKERS` (5.15)
Set max io-wq workers per category (bounded/unbounded).

## Event/Notification

### `IORING_REGISTER_EVENTFD` / `IORING_UNREGISTER_EVENTFD` (5.2)
Register eventfd for CQE notifications. Integrates with epoll/select event loops.

### `IORING_REGISTER_EVENTFD_ASYNC` (5.6)
Only signal eventfd for async completions, not inline ones.

## Miscellaneous

### `IORING_REGISTER_PROBE` (5.6)
Query which opcodes the kernel supports. Essential for portable code.

### `IORING_REGISTER_PERSONALITY` (5.6)
Register credentials for per-SQE identity switching.

### `IORING_REGISTER_RESTRICTIONS` (5.10)
Restrict which opcodes/flags can be used on this ring. Sandboxing.

### `IORING_REGISTER_SYNC_CANCEL` (6.0)
Synchronous cancellation with timeout. Blocks until cancelled.

### `IORING_REGISTER_NAPI` / `IORING_UNREGISTER_NAPI` (6.9)
Register NAPI busy polling. Direct kernel networking stack integration for lowest latency.

### `IORING_REGISTER_CLOCK` (6.10)
Register custom clock source for timeouts.

### `IORING_REGISTER_SEND_MSG_RING` (6.12)
Send `MSG_RING` without having a ring instance. Cross-ring communication from outside.

### `IORING_REGISTER_ZCRX_IFQ` (6.12)
Register network interface queue for zero-copy receive.

### `IORING_REGISTER_MEM_REGION` (6.13)
Register memory region for registered wait arguments.

### `IORING_REGISTER_QUERY` (6.14)
General query interface for various io_uring aspects.

### `IORING_REGISTER_BPF_FILTER` (6.14+)
Register BPF programs for filtering operations.
