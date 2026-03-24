# AppArmor and io_uring

AppArmor has dedicated io_uring mediation since kernel 5.18 (same as SELinux). Ubuntu/Debian default to AppArmor, so this matters.

## Mediated Permissions

Two io_uring-specific permissions in `AA_CLASS_IO_URING`:

| Permission | Maps to | Controls |
|---|---|---|
| `override_creds` | `AA_MAY_APPEND` | Using `IORING_REGISTER_PERSONALITY` to submit ops as different credentials |
| `sqpoll` | `AA_MAY_CREATE` | Creating rings with `IORING_SETUP_SQPOLL` (kernel thread runs under ring owner's creds) |

Basic ring creation and I/O operations don't require special AppArmor rules — only these two elevated capabilities.

## Policy Syntax

```
# Allow all io_uring operations
io_uring,

# Allow only sqpoll
io_uring (sqpoll),

# Allow credential override targeting a specific label
io_uring (override_creds) label=target_profile,

# Deny sqpoll (explicit deny)
deny io_uring (sqpoll),
```

## Default Behavior

- **Ubuntu 22.04+**: Default profiles don't include `io_uring` rules. Unconfined processes work fine. Confined processes (snap packages, custom profiles) get denied.
- **Snap confinement**: No io_uring access by default. Snap interfaces don't expose io_uring permissions.
- **Docker on Ubuntu**: AppArmor profile + seccomp. Both need to allow io_uring.

## Writing Production Profiles

```
profile myserver /usr/bin/myserver {
  # ... standard file/network rules ...

  # Basic io_uring (no SQPOLL, no personality)
  # No rule needed — implicit allow for basic ops

  # SQPOLL: kernel thread does I/O on your behalf
  io_uring (sqpoll),

  # Credential override: rarely needed
  # io_uring (override_creds) label=worker_profile,
}
```

## Comparison with SELinux

| Feature | SELinux | AppArmor |
|---|---|---|
| Permissions | `sqpoll`, `override_creds`, `cmd` | `sqpoll`, `override_creds` |
| Granularity | Type-based (source→target) | Label-based (profile→label) |
| Default distro | RHEL, Fedora | Ubuntu, Debian, SUSE |
| io_uring basic ops | Allowed by default policy | Allowed (no rule needed) |

SELinux has an extra `cmd` permission for `URING_CMD` that AppArmor doesn't mediate separately.

## Debugging

```bash
# Check if AppArmor is denying io_uring
sudo aa-status
dmesg | grep apparmor.*io_uring

# Audit log entries look like:
# apparmor="DENIED" operation="io_uring_setup" class="io_uring"
#   requested_mask="sqpoll" profile="myserver"

# Generate allow rules from denials
sudo aa-logprof
```

## Practical Guidance

1. **Most apps**: No AppArmor rules needed. Basic io_uring just works.
2. **SQPOLL users**: Add `io_uring (sqpoll)` to profile.
3. **Personality users**: Add `io_uring (override_creds) label=<target>`.
4. **Snap packages**: Currently no path to io_uring access.
5. **Containers on Ubuntu**: Need both AppArmor profile update AND seccomp allowlist.

## Source

- AppArmor parser: `parser/io_uring.cc`, `parser/io_uring.h` (Canonical, 2023)
- Kernel LSM hooks: `security/apparmor/lsm.c` — `apparmor_uring_override_creds`, `apparmor_uring_sqpoll`
- Permission constants: `AA_IO_URING_OVERRIDE_CREDS = AA_MAY_APPEND`, `AA_IO_URING_SQPOLL = AA_MAY_CREATE`
