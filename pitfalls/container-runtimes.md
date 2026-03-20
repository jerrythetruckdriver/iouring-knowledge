# io_uring in Container Runtimes

## containerd / CRI-O / runc

Container runtimes (containerd, CRI-O) and the OCI runtime (runc) do **not** use io_uring internally. They manage container lifecycle via Linux namespaces, cgroups, and seccomp — none of which benefit from async I/O.

The io_uring question for containers is about **policy**: do the containers they create get access to io_uring?

## Default Seccomp Profiles

### Docker (Moby) Default Profile

Docker's default seccomp profile **blocks** `io_uring_setup`, `io_uring_enter`, and `io_uring_register`. Applications inside Docker containers cannot use io_uring unless:

1. Run with `--security-opt seccomp=unconfined`
2. Provide a custom seccomp profile that allows io_uring syscalls
3. Run with `--privileged`

### Kubernetes

Kubernetes inherits the container runtime's seccomp default. With containerd + Docker's default profile, io_uring is blocked.

Kubernetes 1.27+ has `SeccompDefault` feature gate (GA). When enabled with the RuntimeDefault profile, io_uring is typically blocked.

To allow io_uring in a pod:

```yaml
apiVersion: v1
kind: Pod
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/allow-io-uring.json
```

### Podman

Podman's default seccomp profile also blocks io_uring syscalls, same as Docker.

## kernel.io_uring_disabled Sysctl

Independent of seccomp, the host kernel's sysctl controls io_uring availability:

| Value | Meaning |
|-------|---------|
| 0 | Enabled for all (default on most distros) |
| 1 | Disabled for unprivileged users |
| 2 | Disabled for everyone |

Containers inherit the host's sysctl value. Even with a permissive seccomp profile, if the host sets `kernel.io_uring_disabled=2`, containers can't use io_uring.

## Why Container Platforms Block io_uring

1. **CVE surface**: io_uring has been responsible for ~60% of kernel exploits via Google's kCTF VRP. Container escape via io_uring bugs is a real threat.
2. **Blast radius**: a kernel bug exploited via io_uring from within a container = full host compromise.
3. **Defense in depth**: most containerized workloads don't need io_uring. Blocking it by default reduces attack surface.

## Enabling io_uring in Containers

### Custom Seccomp Profile

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "syscalls": [
    {
      "names": ["io_uring_setup", "io_uring_enter", "io_uring_register"],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

Merge this with the default profile — don't replace it entirely.

### Docker Run

```bash
docker run --security-opt seccomp=io-uring-profile.json myapp
```

### Kubernetes with AppArmor/Seccomp

Deploy the custom profile to nodes, reference via `seccompProfile.localhostProfile`.

## Risk Assessment

| Environment | Recommendation |
|-------------|----------------|
| Production multi-tenant | Block io_uring (default) |
| Single-tenant, trusted workloads | Allow with custom profile |
| Database containers (ScyllaDB, TigerBeetle) | Allow — they need it for performance |
| CI/CD runners | Block (untrusted code execution) |
| Development/staging | Allow for testing |

## The Trajectory

As io_uring matures and CVE rate drops, container platforms will likely relax defaults. BPF filtering (`IORING_REGISTER_BPF_FILTER`, 6.19) provides fine-grained opcode control that's more practical than all-or-nothing seccomp. Eventually: allow io_uring but restrict to safe opcodes.

We're not there yet.
