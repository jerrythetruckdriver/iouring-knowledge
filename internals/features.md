# Feature Flags

Returned in `io_uring_params.features` after `io_uring_setup()`. Use these to detect kernel capabilities at runtime.

| Flag | Description | Since |
|------|-------------|-------|
| `IORING_FEAT_SINGLE_MMAP` | SQ and CQ share one mmap | 5.4 |
| `IORING_FEAT_NODROP` | Kernel won't drop CQEs on overflow (backpressures instead) | 5.5 |
| `IORING_FEAT_SUBMIT_STABLE` | App can modify SQE data after submit (kernel copies immediately) | 5.5 |
| `IORING_FEAT_RW_CUR_POS` | off=-1 means current file position | 5.6 |
| `IORING_FEAT_CUR_PERSONALITY` | Use current task credentials if personality=0 | 5.6 |
| `IORING_FEAT_FAST_POLL` | Internal poll for async ops (no separate poll SQE needed) | 5.7 |
| `IORING_FEAT_POLL_32BITS` | 32-bit poll events | 5.9 |
| `IORING_FEAT_SQPOLL_NONFIXED` | SQPOLL works without fixed files | 5.11 |
| `IORING_FEAT_EXT_ARG` | Extended `io_uring_enter` argument (timeout + sigmask) | 5.11 |
| `IORING_FEAT_NATIVE_WORKERS` | io-wq uses proper kernel threads | 5.12 |
| `IORING_FEAT_RSRC_TAGS` | Tagged resource updates with notifications | 5.13 |
| `IORING_FEAT_CQE_SKIP` | `IOSQE_CQE_SKIP_SUCCESS` support | 5.17 |
| `IORING_FEAT_LINKED_FILE` | Linked SQEs can use file from previous completion | 5.17 |
| `IORING_FEAT_REG_REG_RING` | Register ring fd in ring itself | 6.4 |
| `IORING_FEAT_RECVSEND_BUNDLE` | Bundle send/recv support | 6.10 |
| `IORING_FEAT_MIN_TIMEOUT` | Minimum wait timeout support in `io_uring_enter` | 6.12 |
| `IORING_FEAT_RW_ATTR` | Read/write attribute support (PI, etc) | 6.13 |
| `IORING_FEAT_NO_IOWAIT` | `IORING_ENTER_NO_IOWAIT` support | 6.13 |

## Feature Detection Strategy

```c
struct io_uring_params p = { .flags = desired_flags };
int fd = io_uring_setup(entries, &p);

if (p.features & IORING_FEAT_FAST_POLL) {
    // Kernel handles poll internally, no need for separate poll SQEs
}

if (p.features & IORING_FEAT_NODROP) {
    // CQEs won't be silently dropped
}
```

For opcode support detection, use `IORING_REGISTER_PROBE`:

```c
struct io_uring_probe *probe = io_uring_get_probe_ring(&ring);
if (io_uring_opcode_supported(probe, IORING_OP_SEND_ZC)) {
    // Zero-copy send available
}
```

Always probe. Never assume based on kernel version alone.
