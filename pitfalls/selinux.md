# SELinux and io_uring

## TL;DR

SELinux has dedicated io_uring access controls since kernel 5.18. Three permissions: `sqpoll`, `override_creds`, `cmd`. Most policies deny all three by default.

## SELinux io_uring Permissions

Added in kernel 5.18 (commit by Paul Moore):

```
# In SELinux policy (type enforcement)
allow my_domain self:io_uring { sqpoll override_creds cmd };
```

| Permission | What it gates | Why restricted |
|-----------|---------------|----------------|
| `sqpoll` | IORING_SETUP_SQPOLL | Kernel thread runs as ring owner — blurs process boundaries |
| `override_creds` | IORING_REGISTER_PERSONALITY | Credential switching via personality IDs |
| `cmd` | IORING_OP_URING_CMD | Direct device commands (NVMe passthrough, socket ops) |

## Default Policy State

Most distributions:

| Distro | SELinux status | io_uring policy |
|--------|---------------|-----------------|
| RHEL 9 | Enforcing by default | io_uring syscall **disabled** via `kernel.io_uring_disabled=2` |
| Fedora | Enforcing by default | io_uring allowed but sqpoll/override_creds/cmd denied for most domains |
| CentOS Stream 9 | Enforcing | Same as RHEL 9 |
| Ubuntu | AppArmor (not SELinux) | No io_uring-specific policy |
| Android | SELinux enforcing | io_uring blocked via seccomp + SELinux |

RHEL 9's approach is belt-and-suspenders: sysctl disables io_uring entirely, AND SELinux would block advanced features even if re-enabled.

## Writing Custom Policy

For production io_uring usage on SELinux-enforcing systems:

```
# my_iouring_app.te

policy_module(my_iouring_app, 1.0)

type my_app_t;
type my_app_exec_t;

# Basic io_uring (no special permissions needed for ring creation)
# io_uring_setup, io_uring_enter, io_uring_register are allowed
# by default if the process has normal file access

# Allow SQPOLL (kernel polling thread)
allow my_app_t self:io_uring sqpoll;

# Allow URING_CMD (NVMe passthrough, socket ops)
allow my_app_t self:io_uring cmd;

# Usually NOT needed — personality/credential switching
# allow my_app_t self:io_uring override_creds;
```

Compile and install:
```bash
checkmodule -M -m -o my_iouring_app.mod my_iouring_app.te
semodule_package -o my_iouring_app.pp -m my_iouring_app.mod
semodule -i my_iouring_app.pp
```

## Audit Log Messages

When SELinux denies io_uring operations:

```
type=AVC msg=audit(...): avc:  denied  { sqpoll } for  pid=1234
  comm="my_server" scontext=system_u:system_r:my_app_t:s0
  tcontext=system_u:system_r:my_app_t:s0 tclass=io_uring permissive=0
```

Use `audit2allow` to generate policy from denials:
```bash
ausearch -m avc -ts recent | audit2allow -M my_iouring_fix
semodule -i my_iouring_fix.pp
```

## Security Layers

io_uring on SELinux systems has up to four security layers:

1. **sysctl** — `kernel.io_uring_disabled` (0=enabled, 1=unprivileged disabled, 2=disabled)
2. **SELinux** — `io_uring { sqpoll override_creds cmd }` permissions
3. **seccomp** — syscall filter (blocks io_uring_setup/enter/register)
4. **Ring restrictions** — Per-ring IORING_REGISTER_RESTRICTIONS + BPF filtering

For RHEL 9, you need to clear all four:
```bash
# 1. Re-enable io_uring
sysctl -w kernel.io_uring_disabled=0

# 2. Add SELinux policy (see above)

# 3. Ensure seccomp profile allows io_uring syscalls (425, 426, 427)

# 4. Ring-level restrictions are opt-in (no action needed)
```

## Recommendation

- **Development:** Disable SELinux (`setenforce 0`) or use permissive mode
- **Production:** Write targeted policy allowing only what your app needs
- **RHEL 9:** Reconsider whether you actually need io_uring — RHEL disabled it for a reason (CVE surface). If you do, accept the risk and enable explicitly
