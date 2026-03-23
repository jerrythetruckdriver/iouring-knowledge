# Application Versioning and Graceful Fallback

## The Problem

io_uring features arrive per-kernel-release. Your application might run on 5.10, 6.1, or 6.19. Hardcoding feature expectations is fragile. You need runtime detection and graceful degradation.

## Detection Strategies

### 1. Probe API (Pre-6.19)

```c
struct io_uring_probe *probe = io_uring_get_probe_ring(&ring);
if (probe && io_uring_opcode_supported(probe, IORING_OP_RECV_ZC))
    use_zc_recv = true;
io_uring_free_probe(probe);
```

Limitations: only tells you if an opcode exists, not if specific flags or features work.

### 2. REGISTER_QUERY (6.19+)

```c
struct io_uring_query_opcodes qo = { .nr = IORING_OP_LAST };
struct io_uring_query_op_info info[IORING_OP_LAST];
qo.ops = (__u64)info;

io_uring_register(fd, IORING_REGISTER_QUERY, &qo, IO_URING_QUERY_OPCODES);
// Now info[IORING_OP_RECV_ZC].flags tells you exactly what's supported
```

The unified query API. Replaces scattered probe/features/trial-and-error. But only available on 6.19+, so you still need fallback for older kernels.

### 3. Feature Flags

```c
struct io_uring_params p = {};
io_uring_queue_init_params(4096, &ring, &p);

if (p.features & IORING_FEAT_RECVSEND_BUNDLE)   use_bundles = true;
if (p.features & IORING_FEAT_MIN_TIMEOUT)        use_min_wait = true;
if (p.features & IORING_FEAT_NO_IOWAIT)          use_no_iowait = true;
if (p.features & IORING_FEAT_RW_ATTR)            use_rw_attr = true;
```

Feature flags tell you about ring capabilities, not individual opcode support.

### 4. Trial and Error

Submit an SQE, check if CQE returns `-EINVAL` or `-EOPNOTSUPP`.

```c
io_uring_prep_recv(sqe, fd, buf, len, 0);
sqe->ioprio |= IORING_RECV_MULTISHOT;
// If CQE res == -EINVAL, multishot recv not supported → fall back to single-shot
```

Ugly but reliable. Use at initialization on a test fd, not in the hot path.

## Fallback Architecture

### Tiered Feature Sets

```c
enum io_backend {
    BACKEND_IOURING_FULL,     // 6.13+: multishot, bundles, registered wait
    BACKEND_IOURING_MODERN,   // 6.0+: multishot accept/recv, provided buffer rings
    BACKEND_IOURING_BASIC,    // 5.10+: basic send/recv, fixed files
    BACKEND_EPOLL,            // fallback
};

enum io_backend detect_backend(void) {
    struct io_uring ring;
    struct io_uring_params p = {};
    
    if (io_uring_queue_init_params(64, &ring, &p) < 0)
        return BACKEND_EPOLL;
    
    struct io_uring_probe *probe = io_uring_get_probe_ring(&ring);
    if (!probe) {
        io_uring_queue_exit(&ring);
        return BACKEND_IOURING_BASIC;
    }
    
    enum io_backend backend = BACKEND_IOURING_BASIC;
    
    if (io_uring_opcode_supported(probe, IORING_OP_RECV) &&
        (p.features & IORING_FEAT_RECVSEND_BUNDLE))
        backend = BACKEND_IOURING_FULL;
    else if (io_uring_opcode_supported(probe, IORING_OP_RECV))
        backend = BACKEND_IOURING_MODERN;
    
    io_uring_free_probe(probe);
    io_uring_queue_exit(&ring);
    return backend;
}
```

### Setup Flag Fallback

```c
uint32_t flags = IORING_SETUP_COOP_TASKRUN | IORING_SETUP_SINGLE_ISSUER
               | IORING_SETUP_NO_SQARRAY;

// Try ideal config first
if (io_uring_queue_init_params(entries, &ring, &(struct io_uring_params){.flags = flags}) < 0) {
    // Drop NO_SQARRAY (pre-6.7)
    flags &= ~IORING_SETUP_NO_SQARRAY;
    if (io_uring_queue_init_params(entries, &ring, &(struct io_uring_params){.flags = flags}) < 0) {
        // Drop COOP_TASKRUN + SINGLE_ISSUER (pre-5.19)
        flags = 0;
        io_uring_queue_init(entries, &ring, 0);
    }
}
```

## Kernel Version → Feature Matrix (Quick Reference)

| Feature | Minimum Kernel |
|---------|---------------|
| Basic io_uring | 5.1 |
| Fixed files/buffers | 5.1 |
| SQPOLL | 5.1 |
| Multishot accept | 5.19 |
| Provided buffer rings | 5.19 |
| COOP_TASKRUN | 5.19 |
| SINGLE_ISSUER | 6.0 |
| Multishot recv | 6.0 |
| Zero-copy send | 6.0 |
| SEND_ZC | 6.0 |
| NO_SQARRAY | 6.7 |
| Futex ops | 6.7 |
| Bundle recv/send | 6.10 |
| Ring resize | 6.13 |
| Registered wait | 6.13 |
| Hybrid IOPOLL | 6.13 |
| BIND/LISTEN | 6.14 |
| EPOLL_WAIT | 6.15 |
| Zero-copy recv (zcrx) | 6.15 |
| Mixed CQEs | 6.18 |
| Mixed SQEs | 6.19 |
| REGISTER_QUERY | 6.19 |

## liburing Version Compatibility

liburing is backward-compatible. Newer liburing on older kernels: prep helpers work, kernel returns `-EINVAL` for unsupported ops. Older liburing on newer kernels: works, just can't access new features via helpers (use raw SQE fields).

**Recommendation**: Compile against latest liburing, detect at runtime. Ship a vendored copy if distro liburing is too old.

## Anti-Patterns

- **Checking kernel version strings** — Distro kernels backport features. Ubuntu 22.04's 5.15 kernel has features not in upstream 5.15.
- **Assuming features by opcode count** — Opcodes can exist but be restricted (seccomp, sysctl, BPF filter).
- **Failing hard on missing features** — Always have a degraded-but-functional path.
- **Probing in the hot path** — Detect once at startup, store in a capabilities struct.
