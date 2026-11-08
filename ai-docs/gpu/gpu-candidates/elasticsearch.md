# Elasticsearch: GPU construction with ordinary CPU search

**Priority: secondary for storage, merging, and integration into an existing engine.** The GPU accelerates graph construction; online search stays on the CPU. Checked on October 28, 2026.

## Data path and architectural idea

Elasticsearch passes vectors to cuVS, builds CAGRA, and writes the result in a compatible HNSW format. The search side can therefore continue reading an ordinary index. This differs from placing the entire active search index in VRAM. [Elastic's integration explanation](https://www.elastic.co/search-labs/blog/elasticsearch-gpu-accelerated-vector-indexing-nvidia).

The inspected writer creates a `CagraIndex`, handles Lucene vector values, writes the graph, and has a GPU merge path. During merging, it acquires a GPU resource before creating a potentially large intermediate copy. This bounds simultaneously live buffers as well as running GPU kernels. [Writer, commit `d8e5eea`](https://github.com/elastic/elasticsearch/blob/d8e5eea42789c6f55bc229fc8ab94d00e41b0923/libs/gpu-codec/src/main/java/org/elasticsearch/gpu/codec/ES92GpuHnswVectorsWriter.java).

**Useful principle for SereneDB:** resource limits must cover data preparation before GPU dispatch. If every waiting query already holds a large staging buffer, a bounded GPU queue alone does not bound RAM.

## Published 2026 benchmark

The September 25, 2026 publication uses 138M × 1024 from MS MARCO and a server with two AMD EPYC 9555 CPUs and **8 × RTX PRO 6000**. It compares the same server with GPUs enabled and disabled, using Rally.

| Metric | CPU | GPU build |
|---|---:|---:|
| Indexing | about 1 hour | under 10 minutes; reported 7× throughput |
| p90 CPU search during concurrent indexing | 45 ms | 7 ms |
| Force merge to four segments/shard | about 4 hours | about 5 minutes |
| Target recall | 0.95 | 0.95 |

The lower search latency comes from freeing the CPU from construction, not GPU query execution. [Elastic results](https://www.elastic.co/search-labs/blog/vector-indexing-gpu-elasticsearch-cuvs).

## Reproducibility

Published Rally settings include eight containerized data nodes, one GPU per node, eight shards, replicas=0, `mapping_type=vectors-only`, refresh=-1 during bulk loading, 64 indexing clients, and bulk=2000. Recall measurement, a `num_candidates` sweep, and separate concurrent load are included. [Experimental configuration](https://github.com/elastic/rally-tracks/issues/1188).

This is stronger evidence than a bare speedup multiplier, but the conditions matter. The result does not describe indexing every field of a multimodal corpus with ordinary visibility-refresh settings. Eight devices are not equivalent to the thesis testbed's single GPU. Force merge is a separate stage and must not silently be included on one side of a comparison and excluded on the other.

Exact reproduction still requires pinning the Elasticsearch/cuVS build, track revision, data, storage characteristics, and container CPU allocation. The material provides a useful basis; no local replay was performed.

## Operational constraints

The inspected documentation requires an Enterprise subscription, Linux x86_64, a supported NVIDIA GPU, and compatible CUDA/cuVS runtimes. GPU detection and forced-enablement checks are provided; an unsupported environment can prevent this mode from starting. [GPU indexing setup](https://www.elastic.co/docs/solutions/search/vector/gpu-vector-indexing).

## Relevance and effort for the thesis

**Estimate:** understanding the existing GPU-build pipeline is medium effort; porting a compatible index writer/merge is high effort. This is not a ready-made SereneDB IVF patch: CAGRA/HNSW and IResearch IVF have different structures.

A useful additional metric is background work's impact on existing search latency. The GPU can benefit the system even if search itself does not become faster. Describe that result accurately as “resource isolation during indexing.”

GPU-build/CPU-serve is relevant to the initial construction experiment. The subsequent resident GPU-search stage remains the required thesis result. Reuse separate ingest/build/merge/search timers and post-construction recall checks.
