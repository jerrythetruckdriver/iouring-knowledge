# Splice Patterns for Video Streaming

## The Problem

Video streaming servers send large files (GB+) to many concurrent clients. Traditional read+send copies data twice: disk → page cache → user buffer → socket buffer. Splice eliminates user-space copies.

## io_uring Splice Chain

```
PIPE(O_NONBLOCK)                           → creates pipe [read_fd, write_fd]
SPLICE(file_fd → pipe_write_fd, len)       → disk → pipe (zero-copy from page cache)
SPLICE(pipe_read_fd → socket_fd, len)      → pipe → socket (zero-copy to network)
```

Link all three for single-submission, fully async file→network transfer:

```c
// SQE 0: create pipe (6.14)
io_uring_prep_pipe(sqe0, IORING_FILE_INDEX_ALLOC);
sqe0->flags = IOSQE_IO_LINK;

// SQE 1: file → pipe
io_uring_prep_splice(sqe1, file_fd, file_offset, pipe_wr, -1, chunk_size, 0);
sqe1->flags = IOSQE_IO_LINK;

// SQE 2: pipe → socket
io_uring_prep_splice(sqe2, pipe_rd, -1, socket_fd, -1, chunk_size, 0);
```

## Chunked Streaming Pattern

Video streaming sends in chunks (HTTP chunked transfer, HLS segments, DASH):

```
For each chunk:
  SPLICE(file → pipe, chunk_size)  ──LINK──  SPLICE(pipe → socket, chunk_size)

Submit multiple chunk pairs in one batch:
  [splice_in, splice_out, splice_in, splice_out, ...]
     LINK          LINK       LINK

Each pair is linked. Pairs are independent (no link between them = parallel).
```

**Pipe reuse:** Create one pipe per connection. Reuse across chunks. Don't recreate per-chunk.

## vs SEND_ZC

| Method | Copies | Syscalls | Best For |
|--------|--------|----------|----------|
| read + send | 2 | 2 per chunk | Nothing (legacy) |
| splice (file→pipe→sock) | 0 (page cache) | 0 (io_uring batched) | Large sequential file serving |
| SEND_ZC + registered buffer | 0 (registered mem) | 0 | Data already in memory |
| SENDMSG_ZC vectored | 0 | 0 | Scatter-gather from multiple buffers |

**Splice wins** when data is on disk and you don't need to inspect it. The page cache provides the zero-copy — data goes from page cache directly to the NIC via DMA scatter-gather.

**SEND_ZC wins** when data is already in registered buffers (transcoded, cached in userspace).

## TEE for Multi-Client

`IORING_OP_TEE` duplicates pipe data without consuming it:

```
File → pipe_A (splice)
  pipe_A → pipe_B (tee)   → socket_1 (splice)
  pipe_A → pipe_C (tee)   → socket_2 (splice)
  pipe_A → socket_0 (splice, consumes)
```

All clients get the same data, zero-copy. Useful for live streaming to multiple viewers of the same content.

## Adaptive Bitrate

For ABR streaming (HLS/DASH), the server selects different quality files per segment:

```
Client requests /segment_42_720p.ts
  → SPLICE(segment_42_720p.ts → pipe → socket)

Client requests /segment_43_1080p.ts (bandwidth improved)
  → SPLICE(segment_43_1080p.ts → pipe → socket)
```

No change to the io_uring pattern. File fd changes per segment, splice chain stays the same.

## Pipe Sizing

Default pipe capacity: 16 pages (64 KB). For video streaming:

```c
fcntl(pipe_fd, F_SETPIPE_SZ, 1048576);  // 1 MB pipe
```

Larger pipe = larger splice chunks = fewer SQEs. Match pipe size to your chunk size.

**io_uring can't set pipe size async** — no `IORING_OP_FCNTL`. Do it once at pipe creation via syscall, or accept the default and use smaller chunks.

## Performance Notes

- Splice throughput is limited by page cache speed, not io_uring
- For sequential large files, the kernel's readahead handles prefetching
- `FADVISE(SEQUENTIAL)` via `IORING_OP_FADVISE` hints the readahead engine
- For random access (seeking within video), splice is less effective — page faults dominate
- O_DIRECT + registered buffers may beat splice for random-access patterns
