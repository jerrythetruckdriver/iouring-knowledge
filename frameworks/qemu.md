# QEMU

Virtual machine monitor. Uses io_uring for guest block device I/O.

## Status: Production (since QEMU 5.0, 2020)

io_uring is a supported block I/O backend alongside linux-aio and thread-pool. Enabled with `-drive aio=io_uring`.

## Architecture

From `block/io_uring.c`:

```c
typedef struct {
    Coroutine *co;      // QEMU coroutine for async
    QEMUIOVector *qiov; // scatter-gather I/O vector
    uint64_t offset;
    ssize_t ret;
    int type;           // READ, WRITE, FLUSH, ZONE_APPEND
    int fd;
    BdrvRequestFlags flags;
    int total_read;     // for short read resubmission
    QEMUIOVector resubmit_qiov;
    CqeHandler cqe_handler;
} LuringRequest;
```

## Supported Operations

```c
QEMU_AIO_WRITE  → io_uring_prep_write / io_uring_prep_writev / io_uring_prep_writev2
QEMU_AIO_READ   → io_uring_prep_read / io_uring_prep_readv
QEMU_AIO_FLUSH  → io_uring_prep_fsync(IORING_FSYNC_DATASYNC)
QEMU_AIO_ZONE_APPEND → io_uring_prep_writev
```

Deliberately simple. QEMU's block layer only needs basic read/write/flush. No networking ops — those go through QEMU's own network stack.

## Design Decisions

**Coroutine integration**: Each request is a QEMU coroutine. `luring_co_submit()` yields the coroutine, which resumes on CQE completion via `aio_co_wake()`. Clean async model.

**Vectored vs non-vectored**: Single-iov requests use `io_uring_prep_write()` / `io_uring_prep_read()`. Multi-iov uses vectored variants. Comment in source: "The man page says non-vectored is faster than vectored."

**FUA support**: `BDRV_REQ_FUA` maps to `RWF_DSYNC` via `io_uring_prep_writev2()`. Conditional on `HAVE_IO_URING_PREP_WRITEV2`. Force Unit Access without separate fsync.

**Short read handling**: `luring_resubmit_short_read()` handles partial reads by resubmitting with adjusted offset/iovec. Short reads are rare but occur on some storage.

**EAGAIN/EINTR retry**: Both are resubmitted automatically. Comment cites a Linux SCSI bug where EAGAIN appears unexpectedly.

## What QEMU Doesn't Use

- No registered buffers (guest memory is managed by QEMU's own allocator)
- No registered files (block devices opened normally)
- No IOPOLL (would require O_DIRECT on the host)
- No SQPOLL (not needed for block I/O rates)
- No NVMe passthrough (QEMU has its own virtio-blk/NVMe emulation)
- No multishot (block ops are one-shot by nature)

## Comparison with linux-aio

QEMU supported linux-aio (`-drive aio=native`) first. io_uring advantages:

| Feature | linux-aio | io_uring |
|---------|-----------|----------|
| Buffered I/O | No (O_DIRECT only) | Yes |
| fsync | Separate syscall | In-ring |
| FUA writes | Not supported | RWF_DSYNC via writev2 |
| Setup | io_setup/io_submit | mmap rings |
| Overhead | ~2 syscalls/op | Batched |

io_uring's buffered I/O support is the key win. It means `-drive cache=writeback` works with async I/O, whereas linux-aio required `-drive cache=none` (O_DIRECT).

## Usage

```bash
# Start VM with io_uring block backend
qemu-system-x86_64 \
    -drive file=disk.qcow2,format=qcow2,aio=io_uring \
    -enable-kvm ...

# Or with raw disk
qemu-system-x86_64 \
    -drive file=/dev/nvme0n1,format=raw,aio=io_uring,cache=none
```

## Performance

io_uring gives QEMU ~10-20% better IOPS for mixed read/write workloads compared to the thread-pool backend (default). The gains come from:

1. Fewer context switches (no thread pool wake)
2. Batched submission (multiple guest I/Os in one `io_uring_enter`)
3. FUA support (avoids separate fsync calls)

For O_DIRECT workloads, io_uring and linux-aio are comparable. io_uring wins on features (FUA, buffered I/O) rather than raw throughput.

## Virtio-blk + io_uring: The Full Stack

```
Guest app → virtio-blk driver → virtqueue → QEMU → io_uring → host kernel → NVMe
```

Two levels of ring-based async I/O. The guest uses virtqueues (structurally similar to io_uring), QEMU translates to io_uring on the host. Each level batches independently.

For maximum performance, use vhost-user-blk (DPDK) or virtiofs with io_uring FUSE backend (6.14+) to bypass QEMU entirely.
