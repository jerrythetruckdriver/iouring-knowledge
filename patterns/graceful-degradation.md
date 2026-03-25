# Graceful Degradation Cookbook

How to write code that uses io_uring when available and falls back cleanly.

## Detection Strategy

Three tiers, from best to worst:

### Tier 1: REGISTER_QUERY (6.19+)

```c
struct io_uring_query_op ops[IORING_OP_LAST];
struct io_uring_query_arg arg = {
    .query = IORING_QUERY_OPCODES,
    .data = (__u64)ops,
    .data_size = sizeof(ops),
};
io_uring_register(&ring, IORING_REGISTER_QUERY, &arg, 1);
// ops[IORING_OP_RECV].flags tells you exactly what's supported
```

### Tier 2: Probe API (5.4+)

```c
struct io_uring_probe *probe = io_uring_get_probe_ring(&ring);
if (io_uring_opcode_supported(probe, IORING_OP_MULTISHOT_ACCEPT))
    use_multishot = true;
io_uring_free_probe(probe);
```

### Tier 3: Trial and Error

```c
// Submit a NOP-like version and check for -EINVAL or -ENOSYS
// Last resort. Works everywhere but ugly.
```

## Feature Matrix Pattern

```c
struct io_features {
    bool has_multishot_accept;
    bool has_multishot_recv;
    bool has_provided_buffers;
    bool has_send_zc;
    bool has_registered_wait;
    bool has_sqpoll;
    bool has_ring_resize;
};

struct io_features detect_features(struct io_uring *ring) {
    struct io_features f = {0};
    struct io_uring_probe *p = io_uring_get_probe_ring(ring);
    if (!p) return f;

    f.has_multishot_accept = io_uring_opcode_supported(p, IORING_OP_ACCEPT);
    f.has_multishot_recv   = io_uring_opcode_supported(p, IORING_OP_RECV);
    f.has_provided_buffers = io_uring_opcode_supported(p, IORING_OP_PROVIDE_BUFFERS);
    f.has_send_zc          = io_uring_opcode_supported(p, IORING_OP_SEND_ZC);
    // Note: probe tells you if the opcode exists, not if specific flags work
    // For flag-level detection, use trial-and-error or REGISTER_QUERY

    f.has_sqpoll = (ring->features & IORING_FEAT_SQPOLL_NONFIXED);
    f.has_ring_resize = true; // try it, catch -EINVAL

    io_uring_free_probe(p);
    return f;
}
```

## Tiered Backend Architecture

```c
typedef struct {
    // Level 0: basic io_uring (5.4+)
    void (*submit_read)(ctx_t *, int fd, void *buf, size_t len);
    void (*submit_write)(ctx_t *, int fd, void *buf, size_t len);

    // Level 1: registered resources (5.4+)
    void (*submit_read_fixed)(ctx_t *, int fd, int buf_idx, size_t len);

    // Level 2: multishot (5.19+)
    void (*submit_accept)(ctx_t *, int listen_fd);
    void (*submit_recv)(ctx_t *, int fd, int buf_group);

    // Level 3: zero-copy (6.0+)
    void (*submit_send_zc)(ctx_t *, int fd, void *buf, size_t len);

    // Level 4: latest (6.13+)
    void (*resize_ring)(ctx_t *, unsigned sq, unsigned cq);
} io_ops_t;

io_ops_t select_backend(struct io_features *f) {
    io_ops_t ops = basic_ops;  // always available

    if (f->has_multishot_recv && f->has_provided_buffers)
        ops = multishot_ops;   // significant upgrade

    if (f->has_send_zc)
        ops.submit_send_zc = send_zc_impl;

    return ops;
}
```

## Setup Flag Fallback Chain

```c
int try_setup(unsigned entries, unsigned *flags_out) {
    // Try best flags first, degrade on failure
    static const unsigned flag_sets[] = {
        // Best: everything
        IORING_SETUP_COOP_TASKRUN | IORING_SETUP_SINGLE_ISSUER |
        IORING_SETUP_DEFER_TASKRUN | IORING_SETUP_NO_SQARRAY,
        // Good: cooperative without defer
        IORING_SETUP_COOP_TASKRUN | IORING_SETUP_SINGLE_ISSUER |
        IORING_SETUP_NO_SQARRAY,
        // Basic: cooperative only
        IORING_SETUP_COOP_TASKRUN | IORING_SETUP_SINGLE_ISSUER,
        // Minimal: nothing
        0,
    };

    struct io_uring ring;
    for (int i = 0; i < 4; i++) {
        int ret = io_uring_queue_init(entries, &ring, flag_sets[i]);
        if (ret == 0) {
            *flags_out = flag_sets[i];
            return 0;
        }
    }
    return -1;  // io_uring completely unavailable
}
```

## epoll Fallback

For applications that must run on kernels without io_uring:

```c
typedef struct {
    enum { BACKEND_IOURING, BACKEND_EPOLL } type;
    union {
        struct { struct io_uring ring; io_ops_t ops; } uring;
        struct { int epoll_fd; } epoll;
    };
} event_loop_t;

event_loop_t *create_event_loop(void) {
    event_loop_t *el = calloc(1, sizeof(*el));

    // Try io_uring first
    struct io_uring_params p = {0};
    int ret = io_uring_queue_init_params(256, &el->uring.ring, &p);
    if (ret == 0) {
        el->type = BACKEND_IOURING;
        struct io_features f = detect_features(&el->uring.ring);
        el->uring.ops = select_backend(&f);
        return el;
    }

    // Fall back to epoll
    el->type = BACKEND_EPOLL;
    el->epoll.epoll_fd = epoll_create1(EPOLL_CLOEXEC);
    return el;
}
```

## Anti-Patterns

- **Checking kernel version strings** — distros backport features, versions lie
- **Probing in the hot path** — detect once at startup, cache the result
- **All-or-nothing** — use what's available, not "io_uring or bust"
- **Ignoring ENOMEM** — RLIMIT_MEMLOCK may prevent ring creation even on new kernels
- **Assuming SQPOLL works** — requires CAP_SYS_NICE or `IORING_SETUP_SQPOLL` permission
