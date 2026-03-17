# Bundle Operations

## What

Bundles batch multiple buffer-selected operations into a single SQE. Instead of submitting N recv/send operations with N SQEs, one SQE grabs multiple buffers from a provided buffer group and processes them all.

Available since kernel 6.10. Feature flag: `IORING_FEAT_RECVSEND_BUNDLE`.

## Flag

```c
#define IORING_RECVSEND_BUNDLE  (1U << 4)  // sqe->ioprio
```

Set alongside `IOSQE_BUFFER_SELECT` in sqe->flags.

## How It Works

### Recv Bundle

1. Submit `IORING_OP_RECV` with `IORING_RECVSEND_BUNDLE | IORING_RECV_MULTISHOT` in ioprio
2. Kernel receives data and fills as many provided buffers as needed
3. Single CQE reports:
   - `cqe->res` = number of buffers consumed
   - `cqe->flags >> IORING_CQE_BUFFER_SHIFT` = starting buffer ID
   - Buffers are contiguous from the starting ID

### Send Bundle

1. Fill multiple provided buffers with data to send
2. Submit `IORING_OP_SEND` with `IORING_RECVSEND_BUNDLE` in ioprio
3. Kernel sends from all selected buffers in one shot
4. CQE reports how many buffers were consumed

## Why It Matters

Without bundles, high-throughput recv generates one CQE per buffer. At 1M packets/sec with 4KB buffers, that's 1M CQEs/sec just for receives. With bundles, a single CQE can represent dozens of buffers — dramatically reducing CQ pressure.

The math: 64-entry CQ ring previously limited you to 64 in-flight receives. With bundles, each CQE can represent N buffers, so the same ring supports N×64 concurrent data chunks.

## Vectorized Send

```c
#define IORING_SEND_VECTORIZED  (1U << 5)  // sqe->ioprio
```

Added alongside bundles. When set on `IORING_OP_SEND` or `IORING_OP_SEND_ZC`, sqe->addr points to an iovec array instead of a flat buffer. Essentially writev semantics through the send path.

## Multishot + Bundle Pattern

The killer combo for network servers:

```
IORING_OP_RECV + IORING_RECV_MULTISHOT + IORING_RECVSEND_BUNDLE
```

One SQE, indefinite lifetime, batched completions. The kernel keeps receiving and filling buffers until the multishot is cancelled or errors out. Each CQE delivers a batch.

## Constraints

- Requires provided buffer rings (`IORING_REGISTER_PBUF_RING`)
- Buffers in a bundle are always contiguous in the ring
- Application must handle partial bundles (fewer buffers than expected)
- CQ overflow is more consequential — losing one CQE means losing an entire batch
