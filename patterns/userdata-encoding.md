# SQE user_data Encoding Patterns

`user_data` is a 64-bit field set on the SQE and returned verbatim in the CQE. It's your only mechanism to match completions to requests. How you encode it matters.

## Approach 1: Pointer

Simplest. Store a pointer to your request context:

```c
struct request {
    int type;
    int fd;
    void *buffer;
    // ...
};

struct request *req = alloc_request();
sqe->user_data = (uint64_t)req;

// On completion:
struct request *req = (struct request *)cqe->user_data;
```

**Pros**: Zero decoding overhead, access to full context.
**Cons**: Must keep request alive until CQE arrives. UAF risk if you mess up lifetime.

## Approach 2: Packed Integer

Encode operation type + index in 64 bits:

```c
// 8 bits: op type, 24 bits: connection ID, 32 bits: sequence number
#define ENCODE(op, conn, seq) \
    (((uint64_t)(op) << 56) | ((uint64_t)(conn) << 32) | (uint32_t)(seq))
#define DECODE_OP(ud)   ((ud) >> 56)
#define DECODE_CONN(ud) (((ud) >> 32) & 0xFFFFFF)
#define DECODE_SEQ(ud)  ((ud) & 0xFFFFFFFF)

sqe->user_data = ENCODE(OP_RECV, conn_id, seq_num);
```

**Pros**: No pointer lifetime issues. Fast dispatch via op type.
**Cons**: Limited bits per field. Indirect lookup to get full context.

## Approach 3: Slab Index

Use a fixed-size array. Index into it:

```c
#define MAX_INFLIGHT 4096
struct request requests[MAX_INFLIGHT];

int idx = alloc_slot();  // bitmap or free list
requests[idx].type = OP_RECV;
requests[idx].fd = client_fd;
sqe->user_data = idx;

// On completion:
struct request *req = &requests[cqe->user_data];
handle(req);
free_slot(cqe->user_data);
```

**Pros**: Cache-friendly (contiguous array), no malloc, bounded memory.
**Cons**: Fixed capacity. Need free-list management.

## Approach 4: Tagged Pointer

Steal low bits from aligned pointer:

```c
// Assuming 8-byte alignment, 3 low bits are free
#define TAG_BITS 3
#define TAG_MASK ((1UL << TAG_BITS) - 1)

#define TAG_PTR(ptr, tag) ((uint64_t)(ptr) | ((tag) & TAG_MASK))
#define UNTAG_PTR(ud) ((void *)((ud) & ~TAG_MASK))
#define GET_TAG(ud) ((ud) & TAG_MASK)

sqe->user_data = TAG_PTR(req, OP_RECV);  // 3 bits = 8 op types
```

## Approach 5: Generation Counter (Anti-UAF)

Detect stale completions after cancellation:

```c
struct conn {
    uint32_t generation;  // Incremented on close/reuse
    // ...
};

#define ENCODE_GEN(conn_idx, gen) \
    (((uint64_t)(gen) << 32) | (uint32_t)(conn_idx))

// On completion:
uint32_t idx = cqe->user_data & 0xFFFFFFFF;
uint32_t gen = cqe->user_data >> 32;
if (conns[idx].generation != gen) {
    // Stale CQE from cancelled/closed connection. Discard.
    io_uring_cqe_seen(&ring, cqe);
    return;
}
```

## Real-World Examples

| Project | Encoding | Why |
|---|---|---|
| TigerBeetle | `*Completion` pointer | Zig, single-owner semantics, no UAF risk |
| RocksDB | Buffer address as ID | MultiRead: address uniquely identifies request |
| ClickHouse | Buffer address | Same pattern as RocksDB |
| QEMU | `LuringRequest*` pointer | Coroutine-based, lifetime managed by caller |
| Swift NIO | Packed 64-bit (32 fd + 16 regID + 8 type) | Clean dispatch, no pointer lifetime issues |
| libuv | Not applicable | Uses epoll, not io_uring for networking |

## Recommendations

1. **Simple server**: Slab index. Bounded memory, cache-friendly, no UAF.
2. **High-performance**: Pointer with generation counter. Fast access, stale detection.
3. **Multi-op per connection**: Packed integer (op type + connection ID). Clean dispatch.
4. **Mixed workloads**: Tagged pointer if you need both context pointer and op type.

## Anti-Patterns

- **Using fd as user_data**: fd can be reused after close. Two CQEs with same "ID" = confusion.
- **Pointer without lifetime guarantee**: Freeing request struct before CQE arrives = UAF.
- **Zero user_data**: Some libraries use 0 as "no request." Avoid it.
- **RocksDB's poison pattern**: Sets freed request user_data to `0xd5d5d5d5d5d5d5d5` to detect UAF. Defensive, good practice.
