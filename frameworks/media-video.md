# io_uring in Video/Media Processing

## GStreamer

GStreamer does not use io_uring. Its I/O model is GLib's GMainLoop (poll-based) with pipeline-driven data flow. Individual elements use synchronous file/network I/O or V4L2/DRM for hardware access.

Why not:
- GStreamer is cross-platform (Linux, macOS, Windows, Android, iOS)
- Pipeline bottleneck is codec processing, not I/O syscall overhead
- V4L2 camera capture and DRM display output have their own kernel interfaces

## FFmpeg

FFmpeg does not use io_uring. File I/O uses standard `read()`/`write()` through `libavformat`'s `AVIOContext`. Network I/O uses socket APIs.

Why not:
- Same cross-platform constraint
- FFmpeg's bottleneck is encode/decode, not I/O scheduling
- Transcoding pipelines are CPU/GPU-bound, not I/O-bound

## Where io_uring Actually Helps in Media

The I/O-bound media workloads where io_uring matters:

### 1. Media Storage Servers

Serving video files to many concurrent clients. This is a **file I/O + networking** problem, not a codec problem.

- Splice for zero-copy file→socket delivery (`IORING_OP_SPLICE`)
- Multishot accept + recv for connection handling
- Registered buffers for DMA-aligned video chunk delivery
- O_DIRECT + IOPOLL for NVMe storage

This is what CDN edge servers do. Cloudflare and Fastly are the relevant players here, not GStreamer.

### 2. Video Ingest / Recording

Writing high-bitrate streams to storage:

- Linked write+fsync chains for reliable recording
- Registered buffers matching camera DMA buffers
- NVMe passthrough for raw device recording at 4K/8K bitrates

### 3. DMA-BUF + zcrx for GPU Pipelines

The 6.16 DMA-BUF zero-copy receive (`IORING_ZCRX_AREA_DMABUF`) enables:

- Network → GPU buffer without CPU copy
- Relevant for: video streaming to GPU decode, AI inference on video frames, remote rendering

This is nascent. No major media framework uses it yet. But the path from NIC DMA → GPU memory without touching CPU cache is the endgame for high-bandwidth video networking.

### 4. Surveillance / Multi-Camera Systems

Many cameras, many streams, many disk writes simultaneously. Classic I/O multiplexing problem. io_uring's batch submission shines here vs per-stream `write()` syscalls.

## Reality Check

Media frameworks won't adopt io_uring for the same reason most software won't: the bottleneck isn't where io_uring helps. Codec processing dominates media workloads. You optimize codecs with SIMD, GPU offload, and hardware encoders — not by shaving microseconds off `read()`.

io_uring matters for the **infrastructure around** media: storage servers, CDN delivery, DMA pipelines. Not for the processing itself.
