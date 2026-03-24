# io_uring in Container Orchestrators

## Kubernetes

Kubernetes itself doesn't use io_uring. Written in Go (can't), and the control plane is not I/O-bound in the relevant sense.

**Where io_uring matters in K8s**: the workloads inside containers. See [container-runtimes.md](/pitfalls/container-runtimes.md) for seccomp/sysctl configuration.

### Container Runtimes

| Runtime | io_uring Status | Notes |
|---------|----------------|-------|
| containerd (runc) | Blocked by default seccomp | Go, no direct use |
| CRI-O (crun) | Blocked by default seccomp | C, no direct use |
| gVisor (runsc) | Not implemented | Syscall emulation, no io_uring |
| Kata Containers | Works (VM-based) | Full kernel inside VM |
| Firecracker | Works (VM-based) | Guest kernel has io_uring |

**gVisor** is notable: it intercepts all syscalls and emulates them in userspace. io_uring_setup/enter/register are not implemented. Workloads requiring io_uring cannot run on gVisor.

**Kata/Firecracker**: VM-based isolation means the guest kernel handles io_uring independently. The host's io_uring restrictions don't apply inside the VM.

## HashiCorp Nomad

Written in Go. Zero io_uring references in the codebase. Same story as Kubernetes — the orchestrator doesn't need it, and Go can't easily use it.

Nomad's exec and raw_exec drivers launch workloads as processes. Those processes can use io_uring if the host kernel allows it (no seccomp filtering by default in Nomad's isolation model, unlike K8s).

## Apache Mesos

Effectively deprecated. Was written in C++ but never adopted io_uring. Containerization via cgroups/namespaces, no special io_uring handling.

## Docker Compose / Swarm

Same as containerd — Docker's default seccomp profile blocks io_uring syscalls. Override with a custom profile:

```json
{
  "defaultAction": "SCMP_ACT_ALLOW",
  "syscalls": [
    {
      "names": ["io_uring_setup", "io_uring_enter", "io_uring_register"],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

Or disable seccomp entirely (`--security-opt seccomp=unconfined`), which is not recommended for production.

## The Pattern

Container orchestrators don't use io_uring themselves — they're control planes written in Go or similar languages. What matters is whether they **allow** workloads to use io_uring:

| Platform | Default | Override Path |
|----------|---------|---------------|
| Kubernetes (runc) | Blocked (seccomp) | Custom seccomp profile in pod spec |
| Kubernetes (Kata) | Allowed | VM isolation, guest kernel |
| Kubernetes (gVisor) | Not available | Can't override — not implemented |
| Nomad (exec) | Allowed | No seccomp by default |
| Docker | Blocked (seccomp) | Custom seccomp profile or `--privileged` |
| Podman | Blocked (seccomp) | Custom seccomp profile |
| Fly.io | Blocked | Firecracker, but io_uring disabled in guest |
| Railway/Render | Blocked | Shared kernel, security policy |

## Enabling io_uring in Kubernetes

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    seccomp.security.alpha.kubernetes.io/pod: localhost/iouring-allowed.json
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: iouring-allowed.json
```

The profile file must exist on every node at the kubelet's configured seccomp profile root (typically `/var/lib/kubelet/seccomp/`).

## Security Implications

Orchestrators block io_uring for good reason:
- io_uring has the highest CVE density of any kernel subsystem (see [cve-timeline.md](/pitfalls/cve-timeline.md))
- Container escape via io_uring vulnerabilities is a demonstrated attack vector
- Google's kCTF data: ~60% of kernel exploits targeted io_uring

**Recommendation**: Only enable io_uring for workloads that demonstrably need it, in isolated namespaces, with `kernel.io_uring_disabled=2` (disable for unprivileged) as the host default. Use BPF filtering (6.19) to restrict opcodes if available.
