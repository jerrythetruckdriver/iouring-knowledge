# io_uring Support in Major Distro Kernels

Not all kernels are created equal. Distro defaults can cripple io_uring.

## The `kernel.io_uring_disabled` sysctl

Added in 5.19. Controls io_uring availability:

| Value | Effect |
|-------|--------|
| 0 | Enabled for all (default on most distros) |
| 1 | Disabled for unprivileged users |
| 2 | Disabled for everyone |

## Distro Matrix (as of March 2026)

| Distro | Kernel | io_uring Enabled? | Notes |
|--------|--------|-------------------|-------|
| Ubuntu 24.04 LTS | 6.8 | Yes (default) | Full support, not restricted |
| Ubuntu 24.10 | 6.11 | Yes | — |
| Ubuntu 25.04 | 6.14 | Yes | zcrx, epoll_wait, bind/listen available |
| Debian 12 (Bookworm) | 6.1 | Yes | Older feature set, no multishot read |
| Debian 13 (Trixie) | 6.12 | Yes | — |
| Fedora 41 | 6.12 | Yes | — |
| Fedora 42 | 6.14 | Yes | — |
| RHEL 9 | 5.14 | **Disabled (sysctl=2)** | Red Hat disabled it entirely |
| RHEL 10 | 6.12 | TBD | May re-enable with restrictions |
| Arch Linux | Rolling | Yes | Always latest |
| Alpine | 6.12+ | Yes | — |
| Android 15 | 6.6 | **Restricted** | Disabled for apps, kernel-only |

## The Red Hat Situation

RHEL 9 ships with `kernel.io_uring_disabled=2`. Their reasoning:

1. io_uring represented ~60% of kernel exploits in Google's kCTF program (2021-2023)
2. Large attack surface for a feature most RHEL workloads don't use
3. Enterprise customers prioritize stability over performance

This means **any io_uring application deployed on RHEL 9 will fail** unless the sysctl is changed. Check at runtime:

```c
int fd = io_uring_setup(1, &params);
if (fd < 0 && errno == ENOSYS) {
    // io_uring disabled or kernel too old
    fallback_to_epoll();
}
```

## Android Lockdown

Google disabled io_uring for userspace apps in Android 15 (kernel 6.6). Only kernel components and privileged services can use it. Same security rationale as RHEL.

## The Seccomp Problem

Even when `io_uring_disabled=0`, seccomp profiles can block `io_uring_setup`, `io_uring_enter`, and `io_uring_register`. Docker's default seccomp profile **blocks io_uring** (as of early 2026). See [cloud-containers.md](cloud-containers.md).

## Runtime Detection Pattern

```c
#include <liburing.h>

bool io_uring_available(void) {
    struct io_uring ring;
    int ret = io_uring_queue_init(1, &ring, 0);
    if (ret < 0) return false;
    io_uring_queue_exit(&ring);
    return true;
}

// For specific features:
bool has_multishot_accept(void) {
    struct io_uring ring;
    if (io_uring_queue_init(1, &ring, 0) < 0) return false;
    struct io_uring_probe *probe = io_uring_get_probe_ring(&ring);
    bool supported = io_uring_opcode_supported(probe,
                                                IORING_OP_ACCEPT);
    io_uring_free_probe(probe);
    io_uring_queue_exit(&ring);
    return supported;
}
```

Or on 6.19+, use `IORING_REGISTER_QUERY` for a single-call capability dump. See [internals/query-api.md](../internals/query-api.md).

## Recommendations

1. **Always have a fallback** — epoll is universally available
2. **Check at startup, not compile time** — kernel may be updated/downgraded
3. **Document requirements** — "Requires Linux 6.1+ with io_uring enabled"
4. **Test on RHEL** — it's where your enterprise customers are, and it's where io_uring is off
5. **Test in Docker** — default seccomp blocks it
