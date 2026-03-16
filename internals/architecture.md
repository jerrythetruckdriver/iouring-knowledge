# Architecture

## Core Design

io_uring uses two memory-mapped ring buffers shared between kernel and userspace:

```
Userspace                         Kernel
┌──────────┐                    ┌──────────┐
│  SQ Ring  │ ──── mmap ─────► │  SQ Ring  │
│  (tail++) │                   │  (head++) │
├──────────┤                    ├──────────┤
│  SQEs    │ ──── mmap ─────► │  SQEs    │
├──────────┤                    ├──────────┤
│  CQ Ring  │ ◄──── mmap ────── │  CQ Ring  │
│  (head++) │                   │  (tail++) │
└──────────┘                    └──────────┘
```

Userspace writes SQ tail, reads CQ head. Kernel writes CQ tail, reads SQ head. Lock-free producer-consumer on both sides.

## Submission Queue Entry (SQE)

64 bytes (or 128 bytes with `IORING_SETUP_SQE128`):

```c
struct io_uring_sqe {
    __u8    opcode;     // operation type
    __u8    flags;      // IOSQE_ flags
    __u16   ioprio;     // I/O priority
    __s32   fd;         // file descriptor
    __u64   off;        // offset (or addr2)
    __u64   addr;       // buffer pointer (or splice_off_in)
    __u32   len;        // buffer size
    __u32   rw_flags;   // per-op flags (union)
    __u64   user_data;  // returned in CQE
    __u16   buf_index;  // fixed buffer index (or buf_group)
    __u16   personality; // credentials index
    __s32   splice_fd_in; // (or file_index)
    __u64   addr3;      // extended (or cmd[0] for 128b SQEs)
};
```

Key design: `user_data` is your correlation token. Set it to whatever lets you find your completion context.

## Completion Queue Entry (CQE)

16 bytes (or 32 bytes with `IORING_SETUP_CQE32`):

```c
struct io_uring_cqe {
    __u64   user_data;  // copied from SQE
    __s32   res;        // result (like syscall return)
    __u32   flags;      // IORING_CQE_F_* flags
    __u64   big_cqe[];  // extra 16 bytes if CQE32
};
```

CQE flags that matter:
- `IORING_CQE_F_BUFFER` — upper 16 bits of flags = buffer ID
- `IORING_CQE_F_MORE` — more CQEs coming from this SQE (multishot)
- `IORING_CQE_F_SOCK_NONEMPTY` — socket has more data (recv hint)
- `IORING_CQE_F_NOTIF` — zero-copy notification (data is safe to reuse)
- `IORING_CQE_F_BUF_MORE` — buffer partially consumed (incremental)

## Memory Layout & mmap Offsets

```c
#define IORING_OFF_SQ_RING      0ULL           // SQ ring
#define IORING_OFF_CQ_RING      0x8000000ULL   // CQ ring
#define IORING_OFF_SQES         0x10000000ULL  // SQE array
#define IORING_OFF_PBUF_RING    0x80000000ULL  // provided buffer rings
```

Since `IORING_FEAT_SINGLE_MMAP`, SQ and CQ rings share a single mmap. Two mmaps total: one for rings, one for SQEs.

## SQ Ring Flags

Kernel sets these in the SQ ring flags field:

- `IORING_SQ_NEED_WAKEUP` — SQ poll thread sleeping, call `io_uring_enter` with `IORING_ENTER_SQ_WAKEUP`
- `IORING_SQ_CQ_OVERFLOW` — CQ ring overflowed, completions were dropped
- `IORING_SQ_TASKRUN` — task work pending, enter kernel (used with `COOP_TASKRUN`)

## The io_uring_enter() Syscall

This is the only syscall you need in most cases:

- Submit SQEs (pass `to_submit` count)
- Wait for CQEs (pass `min_complete` with `IORING_ENTER_GETEVENTS`)
- Both at once

Flags:
- `IORING_ENTER_GETEVENTS` — wait for completions
- `IORING_ENTER_SQ_WAKEUP` — wake SQPOLL thread
- `IORING_ENTER_SQ_WAIT` — wait for SQ space
- `IORING_ENTER_EXT_ARG` — extended wait with timeout
- `IORING_ENTER_REGISTERED_RING` — use registered ring fd
- `IORING_ENTER_NO_IOWAIT` — don't mark as iowait (6.13+)

## SQ Indirection Array

By default, SQ ring contains indices into the SQE array, not SQEs directly. This lets you reorder submissions without copying SQEs.

With `IORING_SETUP_NO_SQARRAY` (6.6+), the indirection is removed. Direct SQE indexing, less overhead.

With `IORING_SETUP_SQ_REWIND` (6.14+), SQ always reads from index 0. No head/tail tracking. Requires `NO_SQARRAY`.
