# io_uring on WSL2

## TL;DR

WSL2 runs a real Linux kernel. io_uring works — with caveats.

## Kernel Version

WSL2 uses Microsoft's custom kernel, currently based on Linux 6.6.y (`linux-msft-wsl-6.6.y` branch). This means:

- **Available**: Everything up to kernel 6.6 — all core io_uring ops, provided buffer rings (5.19), multishot recv (6.0), FUTEX_WAIT/WAKE (6.7), NO_SQARRAY (6.7)
- **Missing**: Everything after 6.6 — hybrid IOPOLL (6.13), ring resize (6.13), registered wait (6.13), BIND/LISTEN ops (6.14), zero-copy receive (6.15), EPOLL_WAIT op (6.15), mixed CQE/SQE (6.18/6.19)

Microsoft updates the WSL kernel slowly. Don't expect 6.13+ features until WSL ships a newer base kernel.

## Config Status

The WSL2 kernel compiles with `CONFIG_IO_URING=y`. It's enabled, not a module.

`kernel.io_uring_disabled` sysctl defaults to 0 (enabled) on WSL2.

## Known Limitations

### No IOPOLL

WSL2's virtual block device (9P/Hyper-V) doesn't support polling. `IORING_SETUP_IOPOLL` will return `-EINVAL` or silently fall back to interrupt-driven I/O depending on the backend.

### No Real SQPOLL Benefit

SQPOLL reduces syscalls. In a VM with Hyper-V overhead, the syscall cost is already dominated by VM exit/entry. SQPOLL still works but the expected performance delta is smaller.

### Networking

WSL2 networking goes through Hyper-V virtual switch. io_uring networking ops work correctly, but you're measuring virtualized NIC performance, not bare metal.

### No NVMe Passthrough

No access to physical NVMe devices. `IORING_OP_URING_CMD` for NVMe requires `/dev/ng*` character devices that don't exist in WSL2.

### No NAPI Busy Poll

Virtual NICs don't support NAPI configuration. `IORING_REGISTER_NAPI` will fail.

## Detection Pattern

```c
#include <sys/utsname.h>
#include <string.h>

int is_wsl2(void) {
    struct utsname buf;
    if (uname(&buf) == 0)
        return strstr(buf.release, "microsoft") != NULL;
    return 0;
}
```

## Practical Advice

WSL2 is fine for **developing** io_uring applications. It's not a valid **benchmark** environment.

Write your code on WSL2, test correctness there, benchmark on bare metal Linux. The io_uring API surface is the same — the performance characteristics are not.

If you need bleeding-edge io_uring features, consider a native Linux VM or dual-boot. WSL2's kernel lags upstream by 1-2 years.
