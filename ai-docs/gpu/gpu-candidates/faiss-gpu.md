# FAISS GPU: the main compute backend

**Priority: primary.** The library directly matches the thesis requirement. The GPU performs search as well as index preparation. Checked on October 28, 2026.

## GPU role and data movement

FAISS provides GPU Flat, IVF-Flat, and IVF-PQ. In resident mode, the index stays in VRAM, query vectors arrive from the CPU or the same device, and IDs and distances are returned. The interface supports CPU/GPU copying of supported FAISS indexes, but GPU variants do not implement the entire CPU API. [FAISS on the GPU](https://github.com/facebookresearch/faiss/wiki/Faiss-on-the-GPU).

For SereneDB, the compatibility boundary matters: the existing IVF belongs to IResearch. The [local reader](../../../iresearch/formats/ivf/ivf_reader.cpp) prepares a `KnnVectorQuery`, and the [writer](../../../iresearch/formats/ivf/ivf_writer.cpp) creates its own index. Having FAISS as a dependency does not make these objects `faiss::IndexIVFFlat` instances.

In the inspected FAISS version, `GpuIndexIVFFlat::copyFrom()` copies the coarse quantizer and inverted lists from a FAISS object; it also selects the classic implementation or cuVS. One verified limitation is that list preallocation through `reserveMemory` is unsupported on this cuVS path. [Code in the project's fork, commit `654e2c8`](https://github.com/serenedb/faiss/blob/654e2c8b88484881b6ab79d3e94c32315e46c5ff/faiss/gpu/GpuIndexIVFFlat.cu).

**Adapter implication:** centroid representation, vector placement in lists, and ID mapping need an explicit contract. Training a separate FAISS index is a possible experiment, but changes the index and must be distinguished from transferring the existing structure.

## Performance evidence

Meta's 2025 comparison uses cuVS Bench: H100, Xeon Platinum 8480CL, 96-dimensional Deep100M, and 5M text vectors with 1536 dimensions. It selects Pareto points at recall@10 = 0.95; latency is measured for one query per call.

| Operation | Classic FAISS GPU | FAISS with cuVS | Speedup |
|---|---:|---:|---:|
| IVF-Flat search, Deep100M | 0.75 ms | 0.39 ms | 1.9× |
| IVF-Flat search, 5M × 1536 | 1.98 ms | 1.14 ms | 1.7× |
| IVF-Flat build, Deep100M | 101.4 s | 37.9 s | 2.7× |

This is **GPU versus GPU**, not a SereneDB result or a complete SQL query. Whether transfers are included must be checked against the pinned harness configuration; the publication's table alone is insufficient. [Meta's methodology and tables](https://engineering.fb.com/2025/05/08/data-infrastructure/accelerating-gpu-indexes-in-faiss-with-nvidia-cuvs/).

The FAISS authors also describe cases where GPUs lose: small indexes and small batches may run better on CPUs because of cache locality and lower overhead. [Comparing GPU vs CPU](https://github.com/facebookresearch/faiss/wiki/Comparing-GPU-vs-CPU).

## Resources and concurrent queries

`StandardGpuResources` provides temporary memory and CUDA streams. Reusing scratch memory helps avoid frequent allocations and synchronization. Interoperation with other CUDA code requires agreement on streams and data readiness. [GPU API](https://github.com/facebookresearch/faiss/wiki/Faiss-on-the-GPU).

FAISS GPU indexes do not guarantee thread safety even for search: temporary resources cannot be shared arbitrarily between simultaneous callers. [Threads and asynchronous calls](https://github.com/facebookresearch/faiss/wiki/Threads-and-asynchronous-calls).

**Our recommendation:** give the first server path a clear owner for the index and resources, define the supported concurrency model, and test it separately. Serializing work through one GPU worker may be an acceptable starting point, but its effects on queuing and p99 must be measured.

## A useful thesis experiment

Compare IResearch CPU, FAISS CPU, classic FAISS GPU, and FAISS-cuVS on one snapshot. For a library comparison, reuse centroids and lists where possible; for an algorithm comparison, plot curves at matched recall. Record preparation, loading, warm calls, and complete server queries separately. Metrics and harnesses are covered in [benchmarks.md](benchmarks.md).

## Implementation effort

**Standalone prototype: low; complete SereneDB adapter: medium/high.** This is an effort estimate, not a measurement. The main costs are converting IResearch data, stable IDs, snapshot lifecycle, memory, and concurrency; the library already implements distances and top-K. GPU compilation is forcibly disabled in the current [third_party/CMakeLists.txt](../../../third_party/CMakeLists.txt); this research did not change the configuration.
