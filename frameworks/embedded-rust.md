# io_uring and Embedded Rust (Embassy, probe-rs)

## Embassy: Wrong Abstraction Layer

[Embassy](https://github.com/embassy-rs/embassy) is an async framework for bare-metal microcontrollers (STM32, nRF52, RP2040, ESP32). It uses Rust's `async/await` with a cooperative executor — no OS, no kernel.

**io_uring is irrelevant to Embassy.** Here's why:

- io_uring is a Linux kernel interface. Embassy runs on bare metal. No kernel, no io_uring.
- Embassy's async model is compile-time task transformation, not kernel-mediated I/O.
- HAL drivers talk directly to hardware registers. There's no syscall boundary to optimize.

The conceptual parallel is interesting though: both Embassy and io_uring solve "how do I efficiently multiplex I/O" — Embassy at the hardware level, io_uring at the kernel level.

## probe-rs

[probe-rs](https://probe.rs/) is a debugging/flashing tool for embedded targets. It runs on the **host** (your dev machine), not on the target. It could theoretically benefit from io_uring for USB/serial I/O, but:

- USB debug probe communication is low-throughput, latency-insensitive
- probe-rs is cross-platform (Windows, macOS, Linux)
- No practical reason to use io_uring here

## Embedded Linux + io_uring

The interesting case is **embedded Linux** — not bare-metal Rust. Think Raspberry Pi, BeagleBone, industrial gateways running Linux.

io_uring works on embedded Linux with standard kernel builds. Relevant considerations:

### ARM32 Considerations

- Memory barriers are explicit (not TSO like x86). liburing handles this correctly.
- libuv disables io_uring on 32-bit ARM (`ifdef __arm__`). If using libuv/Node.js on ARM32, io_uring is off.
- Direct liburing usage works fine on ARM32.

### Resource-Constrained Environments

- Start with small rings (8-16 entries). Use `IORING_REGISTER_RESIZE_RINGS` (6.13+) to grow.
- `IORING_SETUP_COOP_TASKRUN | IORING_SETUP_SINGLE_ISSUER` for minimal kernel overhead.
- Avoid SQPOLL on single-core systems — it steals your only core.
- Registered buffers matter more when RAM is tight (pinned pages, no per-I/O allocation).

### Use Cases

| Application | io_uring Benefit |
|-------------|-----------------|
| Data logger (sensor → storage) | Linked read+write, no syscall per sample |
| Network gateway | Multishot accept+recv, minimal CPU for high connection count |
| Camera/video pipeline | Splice for zero-copy, registered buffers for DMA |
| Industrial control | Low-latency with registered wait, COOP_TASKRUN |

## The Taxonomy

| Environment | io_uring? | Why |
|-------------|-----------|-----|
| Bare-metal Rust (Embassy) | No | No kernel |
| Embedded Linux (RPi, etc.) | Yes | Standard Linux kernel |
| RTOS (Zephyr, FreeRTOS) | No | No Linux kernel |
| Linux + PREEMPT_RT | Yes, with caveats | See `embedded-realtime.md` |

Don't conflate "embedded Rust" with "embedded Linux." They're different worlds.
