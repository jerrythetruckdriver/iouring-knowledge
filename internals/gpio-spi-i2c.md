# io_uring for GPIO/SPI/I2C

## Current State: Not Supported

As of 6.19, there are **no URING_CMD handlers** for GPIO, SPI, or I2C subsystems. These are char devices, but none implement `uring_cmd` in their `file_operations`.

## What Exists

Drivers that implement URING_CMD:

| Subsystem | Driver | Since |
|---|---|---|
| NVMe | `/dev/ng*` | 5.19 |
| Sockets | TCP/UDP | 6.7 |
| FUSE | `/dev/fuse` | 6.14 |
| ublk | `/dev/ublkc*` | 5.19 |

GPIO (`/dev/gpiochip*`), SPI (`/dev/spidev*`), and I2C (`/dev/i2c-*`) use standard `read()`/`write()`/`ioctl()` only.

## Could It Work?

### GPIO

GPIO chardev (libgpiod v2) uses `ioctl()` for line configuration and `read()` on line event fds for edge detection.

- **Edge events**: Could use `IORING_OP_READ` on the GPIO event fd today. Works.
- **Line set/get**: Would need URING_CMD. The GPIO chardev would need to implement `uring_cmd` dispatch for `GPIO_V2_GET_LINE_IOCTL`, `GPIO_V2_LINE_SET_VALUES_IOCTL`, etc.
- **Value**: Low. GPIO operations are microsecond-scale. Async overhead would dominate.

### SPI

SPI transfers via `/dev/spidev*` use `ioctl(SPI_IOC_MESSAGE)`.

- **Bulk transfers**: SPI messages can be large (flash chips, displays). Async offload could help for DMA-backed transfers.
- **Would need**: URING_CMD handler in `spidev.c` that maps `SPI_IOC_MESSAGE` to async transfer submission.
- **Value**: Moderate for high-throughput SPI (e.g., SPI NOR flash). Low for sensor reads.

### I2C

I2C via `/dev/i2c-*` uses `read()`/`write()`/`ioctl(I2C_RDWR)`.

- **Standard read/write**: Already works with `IORING_OP_READ`/`IORING_OP_WRITE` (though I2C drivers may block in the kernel, handled by io-wq).
- **I2C_RDWR transactions**: Would need URING_CMD.
- **Value**: Low. I2C is 100-400 KHz. The bus speed is the bottleneck, not syscall overhead.

## The Pattern for Adding URING_CMD

Any char device can add URING_CMD support by implementing:

```c
static int my_uring_cmd(struct io_uring_cmd *ioucmd, unsigned int issue_flags)
{
    u32 cmd_op = ioucmd->sqe->cmd_op;
    switch (cmd_op) {
    case MY_URING_OP_READ:
        // ... handle async
        io_uring_cmd_done(ioucmd, result, 0, issue_flags);
        return 0;
    default:
        return -EINVAL;
    }
}

static const struct file_operations my_fops = {
    .uring_cmd = my_uring_cmd,
    // ...
};
```

## Practical Recommendation

For embedded Linux with GPIO/SPI/I2C:

1. **GPIO edge events**: Use `IORING_OP_POLL_ADD` on GPIO event fds. This is the async way to wait for GPIO edges without blocking.
2. **SPI/I2C reads**: Use `IORING_OP_READ`/`IORING_OP_WRITE` on the device fds. io-wq handles the blocking.
3. **Batch sensor polling**: Submit multiple READ SQEs to different I2C/SPI device fds in one batch. io-wq parallelizes them across worker threads.

You don't need URING_CMD for basic embedded I/O. Regular ops + io-wq offload handles it. URING_CMD would only matter for zero-copy DMA paths, which these buses don't really support at the userspace level anyway.

## Trajectory

No active kernel patches adding URING_CMD to GPIO/SPI/I2C subsystems. The use case isn't compelling enough — these are low-bandwidth interfaces where the bus speed dominates. URING_CMD development is focused on high-bandwidth paths: NVMe, networking, block devices.
