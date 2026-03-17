# io_uring + eBPF Integration

Two of the most transformative Linux kernel interfaces, converging.

## Three Layers of Integration

### 1. BPF Filtering (IORING_REGISTER_BPF_FILTER, 6.19)

Covered in [pitfalls/bpf-filtering.md](../pitfalls/bpf-filtering.md). Security-focused: restrict which opcodes/flags a ring can use at runtime. The sandboxing story.

### 2. BPF Programs Submitting to io_uring

BPF programs can call `bpf_io_uring_submit()` to push SQEs into a ring from within a BPF program context. This enables:

- **XDP → io_uring pipelines**: packet arrives at NIC, XDP program processes it, submits async I/O via io_uring to store/forward
- **Tracepoint-triggered I/O**: observability program detects condition, triggers async action
- **tc/BPF → io_uring**: traffic control programs that need async follow-up

Status: experimental/in-development. The kernel plumbing exists but the stable API surface is still evolving.

### 3. io_uring as BPF Async Backend

The deeper vision (articulated by Glauber Costa at ScyllaDB, among others): io_uring and eBPF together represent a fundamental shift in how userspace interacts with the kernel.

- **eBPF** lets you push code *into* the kernel
- **io_uring** lets you pull kernel work *out* to userspace asynchronously

Combined: userspace defines policy (BPF), kernel executes it, results flow back via CQ ring. No syscalls in the hot path for either direction.

## The ScyllaDB Vision (2020)

Glauber Costa's "How io_uring and eBPF Will Revolutionize Programming in Linux":

> "Instead of a flow of code that issues syscalls when needed... applications naturally become an event-loop that constantly adds things to a shared buffer, deals with previous entries that completed, rinse, repeat."

The key insight: io_uring isn't just faster I/O. It's a **new programming model** where the kernel becomes an asynchronous coprocessor.

## BPF Iterator for io_uring Introspection

BPF iterators can walk io_uring internal state:
- Enumerate active rings on the system
- Inspect pending SQEs per ring
- Profile CQE latency distributions
- Monitor worker pool utilization

Useful for fleet-wide observability without per-process `/proc` scraping.

## Practical Convergence Points

| Use Case | BPF Side | io_uring Side |
|----------|----------|---------------|
| Network filtering | XDP/tc program | Multishot recv, zcrx |
| Storage scheduling | BPF I/O scheduler | Direct NVMe URING_CMD |
| Security sandbox | BPF filter on ring | IORING_REGISTER_BPF_FILTER |
| Observability | BPF tracepoints | CQE latency tracking |
| Smart NIC offload | XDP processing | Zero-copy receive |

## What's Missing

- Stable `bpf_io_uring_submit()` kfunc API — still in flux
- BPF programs that can *consume* CQEs directly (push model rather than poll)
- Unified BPF + io_uring scheduler for deadline-based I/O prioritization
- Documentation. Almost none exists for the intersection.

## The Trajectory

Short term: BPF filtering for security, BPF tracing for observability.
Medium term: XDP → io_uring pipelines for zero-copy networking.
Long term: BPF as the policy engine, io_uring as the execution engine. The kernel becomes a programmable async I/O coprocessor.

## Sources

- Glauber Costa, "How io_uring and eBPF Will Revolutionize Programming in Linux" (ScyllaDB blog, May 2020)
- io_uring BPF filter patches (kernel 6.19)
- BPF kfunc development discussions on bpf@vger.kernel.org
