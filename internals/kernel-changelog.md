# io_uring Kernel Changelog (6.14–6.19)

Per-release io_uring changes. Only the stuff that matters.

## Linux 6.19 (in development, ~2026)

- **Mixed-size SQEs** (`IORING_SETUP_SQE_MIXED`): 6.18 added mixed CQEs, 6.19 adds the SQE equivalent. Occasional 128b SQE doesn't force all SQEs to 128b.
- **SQ Rewind** (`IORING_SETUP_SQ_REWIND`): Kernel ignores SQ head/tail, fetches SQEs from index 0. Simplifies submission for some use cases.
- **zcrx + SQ/CQ layout queries** (`IORING_REGISTER_QUERY`): Query what zcrx features are available. Both return ring size info for user-provided ring allocation (`IORING_SETUP_NO_MMAP`, `IORING_MEM_REGION_TYPE_USER`).
- **getsockname/getpeername**: `SOCKET_URING_OP_GETSOCKNAME` via `IORING_OP_URING_CMD`. Trivial hookup after net-side refactoring.
- **IORING_REGISTER_ZCRX_CTRL + RQ flushing**: Control interface for zero-copy RX.
- **BPF filter registration** (`IORING_REGISTER_BPF_FILTER = 37`): Register BPF programs to filter io_uring operations. Security/sandboxing primitive.

## Linux 6.18 (~Dec 2025)

- **Mixed-size CQEs** (`IORING_SETUP_CQE_MIXED`): Both 16b and 32b CQEs in the same ring. 32b CQEs get `IORING_CQE_F_32` in flags. Skip CQEs (`IORING_CQE_F_SKIP`) fill gaps at ring wrap.
- **NOP128 and URING_CMD128**: 128-byte opcode variants for mixed SQE rings.

## Linux 6.15 (May 2025)

Big networking release.

- **Zero-copy receive** (`IORING_REGISTER_ZCRX_IFQ`): Bulk receive directly into app memory via hardware NIC queues. No kernel→user copy. Host memory only in this version — DMA-buf planned.
- **epoll_wait via io_uring** (`IORING_OP_EPOLL_WAIT`): Read epoll events through io_uring. Sounds backwards, but enables incremental migration of existing epoll loops to completion-based model.
- **Vectored registered buffers**: Register scatter-gather buffer lists, not just contiguous regions.
- **IORING_OP_READV_FIXED / WRITEV_FIXED**: Vectored I/O with registered (fixed) buffers. Previously you had to choose between vectored or registered — now you get both.

## Linux 6.14 (Mar 2025)

- **FUSE-over-io_uring**: FUSE kernel↔userspace communication via io_uring instead of `/dev/fuse` read/write. Reduces context switches, improves FUSE filesystem performance significantly.
- **R/W integrity/PI metadata** (`IORING_RW_ATTR_FLAG_PI`): New interface to exchange protection information metadata with read/write ops. `attr_ptr` and `attr_type_mask` fields in SQE.
- **IORING_OP_BIND / IORING_OP_LISTEN**: Async socket bind and listen. Completes the async TCP server lifecycle — socket, bind, listen, accept all via io_uring.
- **IORING_OP_PIPE**: Async pipe creation.
- **Memory region registration** (`IORING_REGISTER_MEM_REGION`): Register user-provided memory regions for ring storage.

## Linux 6.13 (Jan 2025)

- **Hybrid IOPOLL** (`IORING_SETUP_HYBRID_IOPOLL`): Combines interrupt-driven and polled I/O. Polls briefly, falls back to interrupts. Better latency than pure interrupt, less CPU than pure poll.
- **Ring resize** (`IORING_REGISTER_RESIZE_RINGS`): Resize CQ ring without destroying the io_uring instance.
- **Clone buffers** (`IORING_REGISTER_CLONE_BUFFERS`): Clone registered buffers from one ring to another.
- **Multishot timeout** (`IORING_TIMEOUT_MULTISHOT`): Recurring timeouts without resubmission.
- **Send vectorized** (`IORING_SEND_VECTORIZED`): SEND/SEND_ZC with io_vec for scatter-gather sends.

## Linux 6.12 (Nov 2024)

- **IORING_OP_RECV_ZC**: Zero-copy receive opcode.
- **Bundle send/recv** (`IORING_RECVSEND_BUNDLE`): Grab multiple buffers from a buffer group in a single operation. CQE returns buffer count + starting buffer ID.
- **Incremental buffer consumption** (`IOU_PBUF_RING_INC`): Provided buffer rings where buffers are consumed incrementally, not fully. Register large buffer ranges, use only what you need.
- **IORING_OP_FIXED_FD_INSTALL**: Install a fixed descriptor into the regular fd table.

## Linux 6.11 (Sep 2024)

- **NAPI busy-poll** (`IORING_REGISTER_NAPI`): Hardware NAPI busy polling integration. Reduces network latency by polling NIC queues directly.
- **Registered clock** (`IORING_REGISTER_CLOCK`): Custom clock source for timeouts.
- **Custom SQ/CQ memory** (`IORING_SETUP_NO_MMAP`): Application provides memory for rings.
- **Send MSG_RING without ring** (`IORING_REGISTER_SEND_MSG_RING`): Send messages between rings via registration, without needing an fd.

## Trends

The trajectory is clear:
1. **Zero-copy everything** — TX landed first, now RX. DMA-buf next.
2. **Fewer syscalls** — SQ_REWIND, registered waits, CQE_SKIP_SUCCESS.
3. **Flexible rings** — Mixed sizes, user-provided memory, resize.
4. **Network completeness** — bind, listen, getsockname. The full socket lifecycle is async now.
5. **Security** — BPF filtering, restrictions API. Sandboxing io_uring properly.
