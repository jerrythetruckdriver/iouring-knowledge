# io_uring and Linux Namespaces

io_uring's interaction with namespaces is subtle and has security implications for containers.

## User Namespaces

`io_uring_setup()` works inside user namespaces. The ring is associated with the creating task's credentials. Key behaviors:

- **File access** follows the task's effective UID/GID within its user namespace
- **SQPOLL threads** inherit the creator's namespace context
- **Personality switching** (REGISTER_PERSONALITY) creates credentials scoped to the ring's namespace
- **io-wq workers** run in the creator's namespace

### The Problem

io_uring operations bypass some namespace-aware permission checks that the VFS layer performs for syscalls. Historically, this has been a source of container escape vulnerabilities. The kernel has progressively tightened this:

- 5.12: io-wq workers became proper kernel threads with correct credentials
- 5.18: Fixed credential inheritance for SQPOLL
- 6.x: Various fixes for namespace-crossing bugs

## PID Namespace

`IORING_OP_WAITID` returns PIDs in the caller's PID namespace, matching waitid(2) semantics. No surprises here.

## Network Namespace

Socket operations use the namespace of the task that created the ring. If you create a ring in netns A, all socket operations happen in netns A — even if the submitting thread has moved to netns B.

This is usually what you want for containers, but can surprise you with SQPOLL (the kernel thread stays in the original netns).

## Mount Namespace

File operations (openat, statx, unlinkat, etc.) use the mount namespace of the task that created the ring. Same "snapshot at creation" semantics as network namespace.

## Cgroup

io_uring work is charged to the cgroup of the task that created the ring, not the task that submits the SQE. With SQPOLL, all I/O is charged to the SQPOLL thread's cgroup (which is the creator's cgroup).

This means cgroup I/O throttling applies correctly for container workloads — as long as the ring was created inside the container.

## kernel.io_uring_disabled Sysctl

The primary namespace-aware control:

| Value | Effect |
|-------|--------|
| 0 | Enabled for all (default on most distros) |
| 1 | Disabled for unprivileged users (non-CAP_SYS_ADMIN) |
| 2 | Disabled for everyone |

RHEL 9 ships with `io_uring_disabled=2`. This is the blunt instrument — it doesn't distinguish between namespaces.

## Container Security Layers

Three defense layers, from coarsest to finest:

### 1. Seccomp (blocks syscalls)

Docker default seccomp profile blocks `io_uring_setup`, `io_uring_enter`, `io_uring_register`. Container can't create rings at all.

### 2. Sysctl (blocks unprivileged)

`kernel.io_uring_disabled=1` prevents non-root users from creating rings. Root inside a container with CAP_SYS_ADMIN can still create them (if user namespace mapping grants the capability).

### 3. Ring Restrictions (limits operations)

`IORING_REGISTER_RESTRICTIONS` limits which opcodes and flags a ring can use. Applied after ring creation. Useful for sandboxing within a container:

```c
struct io_uring_restriction res[] = {
    { .opcode = IORING_RESTRICTION_SQE_OP, .sqe_op = IORING_OP_READ },
    { .opcode = IORING_RESTRICTION_SQE_OP, .sqe_op = IORING_OP_WRITE },
    // Only allow read and write — no socket ops, no file creation
};
io_uring_register(ring_fd, IORING_REGISTER_RESTRICTIONS, res, 2);
```

### 4. BPF Filtering (6.19, finest grain)

`IORING_REGISTER_BPF_FILTER` attaches a BPF program that inspects every SQE. Can make per-operation decisions based on fd, offset, flags, etc.

## Practical Guidance

**Running io_uring in containers:**
1. Custom seccomp profile that allows io_uring syscalls
2. Set `kernel.io_uring_disabled=0` (or 1 for restricted mode)
3. Consider ring restrictions if you're running untrusted code
4. Keep kernel updated — io_uring namespace bugs get fixed regularly

**If you're building a container runtime:**
- Default to blocking io_uring (Docker's approach is correct for general workloads)
- Provide opt-in for performance-sensitive workloads
- Log when io_uring is enabled — it expands the attack surface

## See Also

- [container-runtimes.md](../pitfalls/container-runtimes.md) — Docker/K8s specifics
- [security.md](../pitfalls/security.md) — CVE history and attack surface
- [bpf-filtering.md](../pitfalls/bpf-filtering.md) — BPF-based SQE filtering
- [task-restrictions.md](../internals/task-restrictions.md) — ring restriction API
