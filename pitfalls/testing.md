# Testing io_uring Applications

## liburing Test Suite

The liburing repo ships ~200+ test programs under `test/`. This is the canonical conformance suite for io_uring kernel behavior.

### Running

```bash
cd liburing
make -C test
# Run all tests
cd test && ./runtests.sh
# Run specific test
./test/io_uring-test.t
```

### What It Covers

- **Every opcode**: individual test per operation
- **Edge cases**: CQ overflow, SQ full, short reads, cancelation races
- **Multishot**: accept, recv, read — verify CQE_F_MORE semantics
- **Registration**: buffers, files, pbuf rings, clone, send-msg-ring
- **Advanced features**: SQPOLL, IOPOLL, linked ops, timeouts, futex
- **Regression tests**: named after kernel commits that fixed bugs
- **Platform-specific**: ARM barriers, 32-bit compat, huge pages

### Writing Tests Against liburing

```c
#include <liburing.h>
#include "helpers.h"  // liburing test helpers

static int test_my_feature(void)
{
    struct io_uring ring;
    int ret;

    ret = io_uring_queue_init(8, &ring, 0);
    if (ret) {
        fprintf(stderr, "ring init: %d\n", ret);
        return T_EXIT_FAIL;
    }

    // ... test logic ...

    io_uring_queue_exit(&ring);
    return T_EXIT_PASS;
}
```

Return values: `T_EXIT_PASS` (0), `T_EXIT_FAIL` (1), `T_EXIT_SKIP` (77 — feature not supported on this kernel).

## syzkaller Fuzzing

[syzkaller](https://github.com/google/syzkaller) is the primary kernel fuzzer for io_uring. It has found the majority of io_uring CVEs.

### io_uring syscall descriptions

syzkaller has dedicated descriptions for:
- `io_uring_setup()` — all flag combinations
- `io_uring_enter()` — submit/getevents/registered ring paths
- `io_uring_register()` — all 37+ register opcodes
- SQE generation — random opcode/flag/fd combinations

### Running syzkaller for io_uring

```bash
# Build kernel with KASAN, KCSAN, LOCKDEP, etc.
# syzkaller config snippet:
{
    "enable_syscalls": [
        "io_uring_setup",
        "io_uring_enter",
        "io_uring_register",
        "mmap$IORING_OFF*"
    ]
}
```

### Impact

- Found 60%+ of io_uring CVEs (UAF, double-free, race conditions)
- Jens Axboe runs syzkaller continuously against io_uring-next
- Syzbot dashboard tracks io_uring bugs publicly

## Unit Testing Patterns

### Feature Detection in Tests

```c
static int test_multishot_recv(void)
{
    struct io_uring ring;
    io_uring_queue_init(8, &ring, 0);

    struct io_uring_probe *p = io_uring_get_probe_ring(&ring);
    if (!io_uring_opcode_supported(p, IORING_OP_RECV)) {
        io_uring_free_probe(p);
        io_uring_queue_exit(&ring);
        return T_EXIT_SKIP;  // Don't fail on old kernels
    }
    io_uring_free_probe(p);

    // ... actual test ...
}
```

### CI Considerations

1. **Kernel version matters** — test different kernels in CI
2. **Docker default blocks io_uring** — use `--security-opt seccomp=unconfined`
3. **RHEL disables io_uring** — skip or use sysctl override
4. **Root vs unprivileged** — some features need CAP_SYS_ADMIN
5. **Race conditions** — io_uring bugs often manifest under load; stress tests matter

### Recommended CI Matrix

```
- Ubuntu 24.04 (kernel 6.8) — stable baseline
- Latest mainline (6.17+) — catch regressions
- Kernel with KASAN — memory safety
- Kernel with KCSAN — concurrency bugs
- Unprivileged user — permission model
```

## Benchmarking

For performance testing, use:

- **fio** with `--ioengine=io_uring` — storage benchmarks
- **liburing examples/** — `io_uring-bench`, `io_uring-cp`
- **Custom harness** — for application-specific patterns

See [benchmarks/](../benchmarks/) for methodology and data.

## Sources

- liburing test/ directory (200+ tests)
- syzkaller io_uring syscall descriptions
- syzbot io_uring bug dashboard
