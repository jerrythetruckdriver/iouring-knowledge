# io_uring on Android

## GKI (Generic Kernel Image)

Android uses Google's GKI kernel, based on upstream Linux LTS branches. Current:

- **Android 14**: GKI kernel 6.1
- **Android 15**: GKI kernel 6.6

Both kernels have io_uring compiled in, but **access is restricted**.

## Restrictions

### SELinux

Android's SELinux policy denies `io_uring_setup` for most app contexts:

```
# Typical Android SELinux denial
avc: denied { io_uring_setup } for pid=12345 comm="app_process"
```

Only system-level processes with explicit SELinux policy grants can use io_uring.

### Seccomp (App Sandbox)

The Android app sandbox (zygote-forked processes) applies a seccomp-bpf filter. `io_uring_setup`, `io_uring_enter`, and `io_uring_register` are **blocked** by default.

```
# From Android's syscall filter
io_uring_setup:    SECCOMP_RET_ERRNO(EPERM)
io_uring_enter:    SECCOMP_RET_ERRNO(EPERM)
io_uring_register: SECCOMP_RET_ERRNO(EPERM)
```

### kernel.io_uring_disabled

Some Android vendors set `kernel.io_uring_disabled=2` (disabled for all unprivileged users) in their kernel configs.

## Who Can Use It

| Context | Access |
|---|---|
| Regular Android apps | **Blocked** (seccomp + SELinux) |
| NDK native code in apps | **Blocked** (same sandbox) |
| System services (C++) | **Maybe** (depends on SELinux policy) |
| Root/adb shell | **Yes** (no seccomp, permissive SELinux) |
| Kernel modules | **N/A** (kernel space) |

## Why Google Blocks It

1. **CVE surface**: io_uring had the highest CVE density of any Linux subsystem in 2021-2023. Google's Android security team decided the risk wasn't worth it.
2. **No user-facing benefit**: Android apps use Java/Kotlin I/O through the framework. ART (Android Runtime) doesn't expose io_uring. No app developer is asking for it.
3. **Defense in depth**: Even if io_uring is compiled in, multiple layers (seccomp, SELinux, sysctl) prevent exploitation.

## Android vs Chrome OS

Chrome OS (also Google, also Linux) similarly restricts io_uring via seccomp in the Chrome sandbox. Same rationale.

## For Embedded Android / AOSP

If you build your own AOSP image:
- You can modify the seccomp filter in `bionic/libc/seccomp/`
- You can add SELinux policy grants
- You can set `kernel.io_uring_disabled=0`

This makes sense for single-purpose devices (kiosks, IoT) where you control the app surface.

## Practical Impact

- **App developers**: Don't bother. Use standard `java.nio` or Android's `AsyncTask`/coroutines.
- **Native developers**: Even if you link liburing into your NDK app, the syscalls are blocked.
- **System developers**: If you're building a custom ROM or system service with elevated permissions, io_uring is available but you must fight SELinux.

## Trajectory

Google has shown no interest in enabling io_uring for Android apps. The security team's position is clear: the attack surface reduction outweighs any performance benefit for mobile workloads. This is unlikely to change unless io_uring's CVE rate drops significantly and there's a compelling user-facing use case.
