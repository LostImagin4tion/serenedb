# Milvus: a GPU index inside a database server

**Priority: primary for lifecycle and operational constraints.** Checked on October 28, 2026.

## How the GPU is used

Milvus provides GPU_CAGRA, GPU_IVF_FLAT, GPU_IVF_PQ, and GPU_BRUTE_FORCE. These support GPU search inside the database. CAGRA also offers another mode: build on a GPU, then adapt the index to a CPU. The name GPU_CAGRA alone therefore does not determine the query's execution device. [GPU index overview](https://milvus.io/docs/gpu_index.md).

The documentation separates construction, loading, and search parameters. In particular, `adapt_for_cpu` enables CPU search on a GPU-built index; source-dataset caching parameters affect memory and result refinement. [GPU_CAGRA](https://milvus.io/docs/gpu-cagra.md).

For the thesis, this reinforces the distinction between the logical index, its loaded copy, and execution mode. Residency, device, and copy availability should be visible in diagnostics.

## Implementation evidence

GPU adaptation lives in Knowhere, which Milvus uses. In the pinned code, GPU_IVF_FLAT is registered as a cuVS index node with a thread-pool wrapper; IVF-Flat, IVF-PQ, and CAGRA have separate integration layers. [IVF registration, commit `401c6d1`](https://github.com/zilliztech/knowhere/blob/401c6d195ecdd6f65bcc7d94c13c8e2cf6e5db6e/src/index/gpu_cuvs/gpu_cuvs_ivf_flat.cc), [cuVS bridge](https://github.com/zilliztech/knowhere/blob/401c6d195ecdd6f65bcc7d94c13c8e2cf6e5db6e/src/common/cuvs/integration/ivf_flat_index.cu).

This is a useful example of separating a library backend from the server wrapper. A thread-pool wrapper, however, does not establish thread safety for every shared GPU structure: that depends on the particular resources and implementation.

## Benchmarks and reported gains

Milvus's hybrid CAGRA publication uses NVIDIA L4 and AMD EPYC 7R13 with 16 CPUs, on Cohere1M and Gist1M. With the same target graph degree of 64, the authors report **12–15×** faster CAGRA construction than CPU HNSW. GPU search on the constructed graph provides **5–6×** the throughput of CPU search. [Milvus experiment](https://milvus.io/blog/faster-index-builds-and-scalable-queries-with-gpu-cagra-in-milvus.md).

These figures concern graph-based CAGRA, not IVF-Flat. The publication shows how quality depends on NN-Descent iterations, but its short description does not recover every search/batch/concurrency setting needed for an independently reproducible multiplier. Treat these as vendor results until reproduced.

The documentation itself warns that a GPU index may not reduce latency; maximum throughput needs high load or many query vectors. [Interpretation constraint](https://milvus.io/docs/gpu_index.md).

## Why this is closer to SereneDB than a library benchmark alone

A server must serve many logical indexes and segments, manage loading, and avoid retaining closed copies in VRAM. A GPU may have spare compute capacity while its memory is full of indexes. Milvus is therefore interesting for its build/load/search separation as well as its kernels.

**Our MVP recommendation:** create one GPU copy of a specific snapshot, reuse it, and explicitly invalidate it when source data changes. Confirm that contract first, then consider multi-index eviction or automatic device placement.

Record unsupported metric, operation, and setting combinations for the chosen Milvus/cuVS version. Do not copy numerical limits across documentation versions: even API tables may describe different backends.

## Implementation effort and a useful check

**Estimate:** reusing library-level ideas is medium effort; porting the complete server lifecycle is high effort. The thesis does not need a distributed database, multiple query nodes, or an automatic GPU scheduler.

A useful experiment is loading one and several indexes while measuring steady/peak VRAM, repeated load/unload, concurrent search, and OOM behavior. For performance, compare GPU_IVF_FLAT with the CPU path at matched recall, separately from graph-based CAGRA. See [benchmarks.md](benchmarks.md) for methodology.
