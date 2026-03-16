# Security Considerations

## Attack Surface

io_uring is a large, complex kernel subsystem. It exposes 65+ operations through a shared-memory interface. This means:

1. **Large attack surface** — every opcode is a potential vulnerability vector
2. **Complex state machine** — linked ops, multishot, cancellation create intricate state
3. **Shared memory** — ring buffers are writable by userspace, kernel must validate everything

## CVE History

io_uring has had its share of security issues. Notable categories:

- **Use-after-free** in request handling (multiple CVEs through 2021-2023)
- **Privilege escalation** via file descriptor handling
- **Reference counting bugs** in registered resources
- **Race conditions** in cancellation paths

The attack surface has been the subject of extensive fuzzing (syzkaller finds io_uring bugs regularly).

## Container/Sandbox Restrictions

### Docker
io_uring was blocked in Docker's default seccomp profile until late 2023. `io_uring_setup`, `io_uring_enter`, and `io_uring_register` were all denied.

### Android
Google disabled io_uring for unprivileged processes in Android. The attack surface was deemed too large for the mobile threat model.

### ChromeOS
Disabled io_uring in production kernels.

### gVisor
gVisor (Google's container sandbox) has limited io_uring support.

## Restrictions API

io_uring has a built-in restrictions mechanism:

```c
// Create ring disabled
params.flags = IORING_SETUP_R_DISABLED;
int fd = io_uring_setup(entries, &params);

// Apply restrictions
struct io_uring_restriction res[] = {
    { .opcode = IORING_RESTRICTION_SQE_OP, .sqe_op = IORING_OP_READ },
    { .opcode = IORING_RESTRICTION_SQE_OP, .sqe_op = IORING_OP_WRITE },
    // Only allow read and write
};
io_uring_register(fd, IORING_REGISTER_RESTRICTIONS, res, 2);

// Enable
io_uring_register(fd, IORING_REGISTER_ENABLE_RINGS, NULL, 0);
```

Once restrictions are set and the ring is enabled, they can't be changed.

## BPF Filtering (6.14+)

`IORING_REGISTER_BPF_FILTER` allows attaching BPF programs to filter operations. More flexible than static restrictions.

## Best Practices

1. **Use restrictions** if exposing io_uring to untrusted code
2. **Limit opcodes** to what's actually needed
3. **Keep kernel updated** — io_uring security patches are frequent
4. **Monitor** — io_uring operations aren't always visible in traditional audit frameworks
5. **Probe, don't assume** — graceful degradation when features aren't available

## The Trade-off

io_uring's performance comes from bypassing traditional syscall paths. Those paths existed partly for security enforcement. The trade-off is real: more performance, more attack surface.

For production servers you control, it's worth it. For multi-tenant environments or sandboxes, evaluate carefully.
