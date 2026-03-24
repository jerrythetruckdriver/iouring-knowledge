# Kernel Live Patching and io_uring

## TL;DR

kpatch/livepatch can patch io_uring code while rings are active. It works but has edge cases. io_uring's kernel threads (SQPOLL, io-wq workers) interact with livepatch's consistency model.

## How Livepatch Works

Linux livepatch (kpatch, kGraft, SUSE kLP) replaces kernel function bodies at runtime:

1. New function version loaded as module
2. ftrace hooks redirect calls from old → new function
3. Consistency model ensures all tasks transition atomically
4. Old code becomes unreachable

## io_uring Interaction Points

### SQPOLL Thread

The SQPOLL kernel thread runs in a tight loop calling io_uring submission functions. Livepatch needs this thread to reach a "consistent" state (return to a known stack frame) before patching.

- SQPOLL threads call `schedule()` when idle (sq_thread_idle timeout)
- Livepatch transitions happen at `schedule()` boundaries
- **Problem:** A busy SQPOLL thread that never sleeps delays livepatch transition
- **Mitigation:** SQPOLL always hits `schedule()` eventually (idle timeout, or SQ_NEED_WAKEUP)

### io-wq Workers

io-wq worker threads process offloaded requests. They `schedule()` between requests, so livepatch transitions are straightforward.

### Active Rings

Existing ring mappings (SQ/CQ shared memory) are unaffected by livepatch — they're data structures, not code. The kernel functions operating on those structures get patched.

## CVE Patching Scenario

Most io_uring CVEs are UAF or logic bugs in specific opcode handlers. Livepatch can fix these without ring teardown:

```
# Example: patching io_uring's recv handler
# 1. Livepatch module replaces io_recv() with fixed version
# 2. In-flight recv requests complete with old code
# 3. New recv submissions use patched code
# 4. No ring restart needed
```

## Limitations

1. **Struct layout changes** — Livepatch can't change `struct io_kiocb` or `struct io_ring_ctx` layout. If a CVE fix requires struct changes, a full reboot is needed.

2. **New opcodes** — Can't add opcodes via livepatch (requires userspace-visible ABI changes).

3. **SQPOLL latency spike** — During livepatch transition, SQPOLL thread may need to be forced through `schedule()`, causing a brief latency spike.

4. **Data structure invariants** — If a bug corrupted in-memory ring state, livepatch fixes the code but not the corrupted data. Ring teardown + recreation may still be needed.

## Practical Guidance

- **Most io_uring CVEs** are patchable via livepatch (function-level fixes)
- **Structural bugs** (ring corruption, refcount issues) may need ring restart
- **RHEL/SUSE** ship livepatch for io_uring CVEs when possible
- **io_uring disabled by default on RHEL 9** — no io_uring livepatches needed if disabled

## 6.19 Live Update Orchestrator

Linux 6.19 introduces LUO (Live Update Orchestrator) for kexec-based reboots that preserve VM state. This is complementary to livepatch:

- Livepatch: fix individual functions, no reboot
- LUO: full kernel replacement via kexec, VMs survive

io_uring rings don't survive kexec/LUO — they're kernel-internal state that gets destroyed. Applications need to detect the reboot and recreate rings.
