# Qdrant: GPU HNSW construction through Vulkan

**Priority: secondary as an example of a custom GPU stack and indexing's effect on a server.** Checked on October 28, 2026.

## What runs on the GPU

Qdrant supports GPU HNSW indexing through Vulkan. This is its own path, separate from CUDA/FAISS/cuVS; GPU use as an online ANN-query backend is not established here. A special build/image and GPU configuration are required. [Running with GPU](https://qdrant.tech/documentation/ops-configuration/running-with-gpu/).

The important separation is that the GPU can prepare the graph, after which ordinary search consumes the result. This accelerates indexing/reindexing without moving all search semantics to the device.

## Source-code evidence

The GPU module separates device management, batched points, graph and layer construction, links, vector storage, visited flags, and shader building. It has parallel-group settings and optional half precision. [Module, commit `222b2e4`](https://github.com/qdrant/qdrant/blob/222b2e44ab1c3f17e8246396ead6558e45041df7/lib/segment/src/index/hnsw_index/gpu/mod.rs).

The documentation explicitly states that forced half precision concerns data inside the GPU and does not change the storage type. A correct comparison must therefore record computational precision as well as stored-vector type. [GPU settings](https://qdrant.tech/documentation/ops-configuration/running-with-gpu/).

**Our assessment:** Vulkan portability is useful survey material, but adding a second GPU API to a CUDA thesis needs a separate objective. The value here is the architecture and experiment, rather than reproducing the stack.

## Metrics used to demonstrate gains

The official tutorial compares Qdrant 1.19 on two clusters: 4 vCPUs, 16GB RAM, 64GB disk, identical 100K Cohere Wikipedia vectors, and identical HNSW parameters. Indexing duration is estimated by polling optimization status.

| Metric | CPU cluster | GPU-indexing cluster |
|---|---:|---:|
| HNSW indexing | 66.3 s | about 6.1 s |
| Search load in this run | about 21 QPS | about 21 QPS |
| Maximum observed latency during indexing | about 0.31 s | about 0.07 s |

The authors report similar p50/p95 and different tails. This illustrates an approximately tenfold shorter indexing window, not tenfold faster search. [Complete tutorial and measurement code](https://qdrant.tech/documentation/tutorials-operations/gpu-accelerated-hnsw-indexing/).

## Benchmark limitations

Equal `m` and `ef_construct` do not replace a matched-recall check. The tutorial mainly demonstrates build duration and search under indexing; it does not provide a complete quality Pareto curve. Maximum latency is sensitive to query count and run duration, which differ between CPU/GPU builds. It cannot support a universal p99 conclusion.

About 21 QPS reflects this load generator and query path; it does not establish maximum server capacity. Reproduction needs the GPU model, dimensionality/metric, query count, load-generation method, and a separate recall test.

## Relevance to the thesis

Separate faster kernels from freeing CPU resources for other tasks. A possible SereneDB experiment combines background construction with an independent search client, measuring query latency, error rate, and build duration simultaneously. This is an additional scenario after the main CPU/GPU-search comparison.

Another lesson is to compare resource consumption over the complete lifecycle. A fast build using a temporary GPU and a continuously provisioned resident GPU copy have different economics. Use your own hardware costs or current prices.

## Effort

**Estimate:** reproducing the existing scenario is medium effort; a custom Vulkan HNSW builder is very high effort. The latter is unnecessary for the selected IVF/FAISS MVP. Applicable ideas include a separate GPU lifecycle, a diagnosable execution mode, batch construction, and measuring CPU contention. Turning these ideas into SereneDB numbers requires a server benchmark.
