# Writing Custom URING_CMD Handlers

## What URING_CMD Is

`IORING_OP_URING_CMD` (opcode 46) lets kernel drivers expose custom async operations through io_uring. The driver defines command semantics; io_uring handles the submission/completion lifecycle.

This is the mechanism behind:
- NVMe passthrough (block/nvme)
- Socket options (net/socket)
- FUSE operations (fs/fuse)
- ublk I/O (block/ublk)

## How It Works

### SQE Layout

```c
sqe->opcode = IORING_OP_URING_CMD;
sqe->fd     = file_fd;           // file that implements uring_cmd
sqe->cmd_op = driver_command;    // driver-defined command number
sqe->cmd[0..79] = ...;          // 80 bytes of command-specific data
```

With `IORING_SETUP_SQE128`, the `cmd` field extends to 80 bytes. Without it, only the union overlap area is available.

### Kernel Driver Side

A driver implements the `uring_cmd` file operation:

```c
static const struct file_operations my_fops = {
    .uring_cmd = my_uring_cmd,
    .uring_cmd_iopoll = my_uring_cmd_iopoll,  // optional: for IOPOLL
};

static int my_uring_cmd(struct io_uring_cmd *ioucmd, unsigned int issue_flags)
{
    // Extract command from SQE
    u32 cmd_op = ioucmd->sqe->cmd_op;
    
    // Access command-specific data
    void *cmd = (void *)ioucmd->sqe->cmd;
    
    // If can complete immediately:
    io_uring_cmd_done(ioucmd, result, 0, issue_flags);
    return -EIOCBQUEUED;
    
    // If need async completion:
    // Store ioucmd, complete later with io_uring_cmd_done()
    return -EIOCBQUEUED;
}
```

### Key APIs (Kernel Side)

```c
// Complete a command
void io_uring_cmd_done(struct io_uring_cmd *ioucmd, 
                       ssize_t ret, u64 res2,
                       unsigned issue_flags);

// Access fixed buffers from driver
int io_uring_cmd_import_fixed(u64 ubuf, unsigned long len,
                              int rw, struct iov_iter *iter,
                              void *ioucmd);

// For multishot commands (6.18)
bool io_uring_cmd_complete_in_task(struct io_uring_cmd *ioucmd,
                                   io_uring_cmd_task_work_cb cb);
```

### Fixed Buffer Support

Set `IORING_URING_CMD_FIXED` in `sqe->uring_cmd_flags` and `sqe->buf_index` to use registered buffers. The driver calls `io_uring_cmd_import_fixed()` to map the buffer.

### SQE128 for Large Commands

NVMe needs 64-byte commands. With `IORING_SETUP_SQE128`, each SQE is 128 bytes, giving 80 bytes for the `cmd` payload. Without it, you're limited to the union overlap.

Since 6.18, `IORING_SETUP_SQE_MIXED` lets you use both 64-byte and 128-byte SQEs on the same ring. Only commands that need it pay the cost.

### Multishot URING_CMD (6.18)

A driver can produce multiple CQEs from a single URING_CMD SQE. Primary use case: ublk batch fetch.

```c
// Driver calls io_uring_cmd_done() multiple times
// Sets IORING_CQE_F_MORE on intermediate completions
// Final completion clears the flag
```

## Examples in the Wild

| Driver | Commands | Notes |
|--------|----------|-------|
| NVMe | IO read/write, admin commands | 64-byte NVMe commands via SQE128 |
| Socket | SIOCINQ, GETSOCKOPT, SETSOCKOPT, TX_TIMESTAMP | General socket operations |
| FUSE | FUSE_READ, FUSE_WRITE, etc. | Kernel↔userspace filesystem IPC |
| ublk | IO commands, zero-copy fetch | Userspace block device I/O |

## When to Write Your Own

You have a kernel module or driver that:
- Handles device-specific I/O commands
- Benefits from io_uring's batching and zero-syscall model
- Needs async completion notification
- Wants to share registered buffers with userspace

The pattern is: define your command opcodes, implement `uring_cmd` in your `file_operations`, use `io_uring_cmd_done()` for completion. The io_uring infrastructure handles everything else — queuing, polling, CQE delivery, cancellation.

## Constraints

- Must be a file with `file_operations.uring_cmd` set
- Character devices, block devices, and sockets can implement it
- Regular files cannot (they use standard read/write opcodes)
- Driver must handle `IO_URING_F_NONBLOCK` in issue_flags for inline completion attempts
