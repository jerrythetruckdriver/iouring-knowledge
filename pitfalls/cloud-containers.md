# io_uring in Cloud and Containers

io_uring's attack surface made cloud providers nervous. Here's where things stand.

## The Problem

io_uring has been a CVE magnet. Between 2020-2024, dozens of privilege escalation vulnerabilities. The kernel exposes complex state machines through shared memory — a rich target.

Google's response in 2023: disable io_uring in ChromeOS and production servers. Other cloud providers followed suit for multi-tenant environments.

## Kernel Controls

### sysctl: kernel.io_uring_disabled (6.1+)

```bash
# Check current state
sysctl kernel.io_uring_disabled

# Values:
# 0 = enabled for everyone (default)
# 1 = disabled for unprivileged users
# 2 = disabled for everyone (even root)
sysctl -w kernel.io_uring_disabled=1
```

### seccomp

Container runtimes (Docker, containerd) use seccomp profiles. Default Docker profile **blocks io_uring_setup and io_uring_enter** since Docker 20.10.

```json
{
  "names": ["io_uring_setup", "io_uring_enter", "io_uring_register"],
  "action": "SCMP_ACT_ERRNO",
  "errnoRet": 38
}
```

### IORING_REGISTER_RESTRICTIONS (5.12+)

Fine-grained per-ring restrictions:

```c
struct io_uring_restriction res[] = {
    { .opcode = IORING_RESTRICTION_REGISTER_OP, .register_op = IORING_REGISTER_FILES },
    { .opcode = IORING_RESTRICTION_SQE_OP, .sqe_op = IORING_OP_READ },
    { .opcode = IORING_RESTRICTION_SQE_OP, .sqe_op = IORING_OP_WRITE },
    { .opcode = IORING_RESTRICTION_SQE_FLAGS_REQUIRED, .sqe_flags = IOSQE_FIXED_FILE },
};
io_uring_register(ring_fd, IORING_REGISTER_RESTRICTIONS, res, 4);
io_uring_register(ring_fd, IORING_ENABLE_RINGS, NULL, 0);
```

After enabling restrictions, the ring only accepts whitelisted operations.

### BPF Filtering (6.19+)

`IORING_REGISTER_BPF_FILTER` attaches eBPF programs that inspect and filter SQEs at submission time. More flexible than static restrictions.

## Cloud Provider Status (2025-2026)

| Provider | Default | Notes |
|----------|---------|-------|
| **AWS** | Blocked in many managed services | ECS, Lambda: seccomp blocks. EC2 bare metal: available. EKS: depends on seccomp profile. |
| **GCP** | Blocked by default | GKE: seccomp default blocks it. Compute Engine: available. Cloud Run: blocked. |
| **Azure** | Varies | AKS: seccomp blocks by default. VMs: available. |
| **Cloudflare Workers** | Blocked | V8 isolates, no syscall access. |
| **Fly.io** | Available | Firecracker VMs, full kernel access. |
| **Hetzner** | Available | Bare metal / dedicated VMs. |

## Enabling io_uring in Containers

### Docker

```bash
# Custom seccomp profile allowing io_uring
docker run --security-opt seccomp=custom-profile.json ...

# Or disable seccomp entirely (not recommended for production)
docker run --security-opt seccomp=unconfined ...
```

### Kubernetes

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    seccomp.security.alpha.kubernetes.io/pod: "unconfined"
    # Or use a custom seccomp profile
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/io-uring-allowed.json
```

### systemd

```ini
[Service]
SystemCallFilter=@io-event io_uring_setup io_uring_enter io_uring_register
```

## Detection at Runtime

Your application should detect io_uring availability gracefully:

```c
#include <sys/syscall.h>
#include <unistd.h>

int io_uring_available(void) {
    // Try to create a minimal ring
    struct io_uring_params p = { 0 };
    int fd = syscall(__NR_io_uring_setup, 1, &p);
    if (fd >= 0) {
        close(fd);
        return 1;
    }
    // ENOSYS = syscall doesn't exist (old kernel)
    // EPERM = blocked by seccomp or sysctl
    return 0;
}
```

TigerBeetle distinguishes the failure modes:
- `SystemOutdated` → seccomp blocking the syscall
- `PermissionDenied` → `kernel.io_uring_disabled` sysctl

## Recommendations

1. **Test in your deployment environment** — don't assume io_uring works because it works locally
2. **Always have a fallback** — epoll/poll path for when io_uring is blocked
3. **Use IORING_REGISTER_RESTRICTIONS** — minimize attack surface per ring
4. **Pin kernel versions** — io_uring behavior changes frequently
5. **Monitor kernel.io_uring_disabled** — cloud providers may change defaults
6. **Dedicated VMs > shared containers** — for io_uring-dependent workloads, avoid multi-tenant seccomp restrictions
