# io_uring in ML Inference Pipelines

No major ML framework uses io_uring. Not TensorFlow, not PyTorch, not ONNX Runtime, not TensorRT. Zero io_uring references in any of their codebases.

## Why Not?

ML inference has two I/O patterns:

1. **Model loading** — one-time, large sequential read (hundreds of MB to hundreds of GB). `mmap()` or `read()` in a thread. Not a hot path.
2. **Data pipeline** — batch loading for training/inference. Framework-specific data loaders handle this with thread pools and prefetch queues.

Neither pattern is syscall-bound. The bottleneck is:
- GPU compute (inference kernel execution)
- CPU↔GPU data transfer (PCIe bandwidth)
- Data decoding (image decode, tokenization)
- Memory allocation and tensor layout

Saving 2μs per syscall when your inference takes 5-50ms is noise.

## Where io_uring Could Help

### Storage-Bound Inference Serving

When model serving at scale needs to load many models on-demand (cold start), io_uring could accelerate model file reads:

```
Request → check cache → miss → load model from NVMe → inference → response
                                      ↑
                          io_uring with O_DIRECT + IOPOLL
                          Registered buffers for model weight tensors
                          Linked read chain for multi-file models
```

This matters for:
- **Serverless inference** (Lambda-style, model per request)
- **Multi-tenant serving** (many models, limited GPU memory, frequent swap)
- **RAG pipelines** (loading embedding indices from disk)

### DMA-BUF Zero-Copy (6.16)

The real opportunity. DMA-BUF backed zcrx could enable:
```
NIC → io_uring zcrx → DMA-BUF → GPU memory
```

Skip CPU entirely for network→GPU data transfer. This is the GPU-Direct RDMA path but via io_uring instead of Mellanox verbs. Currently theoretical — no framework implements this.

### Training Data Pipeline

Large-scale training reads TB of data:
```
Storage → decode → preprocess → batch → GPU
```

io_uring could replace the "Storage →" part:
- Provided buffer rings for prefetch buffers
- SQPOLL for zero-syscall reads
- NVMe passthrough for maximum IOPS

But frameworks already solve this with multi-threaded data loaders (PyTorch DataLoader, tf.data). The data pipeline is typically not the bottleneck unless you're training on spinning rust.

## Who Might Adopt First

1. **TigerBeetle** pattern applied to model serving — Zig/Rust serving frameworks with custom I/O
2. **vLLM** or **TGI** — if model loading latency becomes the bottleneck for autoscaling
3. **Triton Inference Server** — NVIDIA could integrate for model repository I/O
4. **Custom inference on NVMe** — companies with large model zoos on local NVMe arrays

## The Reality

ML frameworks are Python-first, cross-platform, and GPU-bottlenecked. Adding a Linux-only I/O optimization to a Python framework provides near-zero user-visible improvement. The DMA-BUF path is the interesting future, but requires GPU driver integration that doesn't exist yet.

If you're building a custom inference server in C/C++/Rust/Zig and your workload is storage-bound (many models, frequent cold starts, NVMe), io_uring is worth it. For everyone else, it's premature optimization.
