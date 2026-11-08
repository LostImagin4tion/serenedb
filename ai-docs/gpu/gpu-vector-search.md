# GPUs for vector search and index construction

[Research index](README.md) · [Candidate profiles](gpu-candidates/README.md) · [Benchmarks](gpu-candidates/benchmarks.md)

This survey covers GPU search, GPU-assisted vector-index construction, filtered ANN, and datasets larger than VRAM. SQL operator offload is covered in the [SQL survey](gpu-sql-offload.md).

Speedup figures come from authors' and vendors' publications; no local measurements were performed for this survey.

The initial construction experiment is guided by **LanceDB and PGPU**, with [CPU-build profiling before GPU offload](gpu-serenedb.md#initial-experiment-gpu-assisted-ivf-construction). The subsequent GPU-query stage draws on **FAISS/cuVS, Milvus, DuckDB FAISS extension, and pg_cuvs**. The latter is particularly useful for studying SQL-integration costs, although it is experimental. RUMMY demonstrates IVF larger than VRAM. Qdrant, Elasticsearch, AWS, and Oracle are useful for GPU construction and index lifecycle.

## FAISS: ready-made GPU indexes and resource management

FAISS provides GPU implementations of Flat, IVF-Flat, and IVF-PQ. An index can remain on the device; `add()` and `search()` accept CPU or GPU memory. A query already on the same GPU needs no extra transfer. An important detail: GPU indexes do not support the entire interface of their CPU counterparts. [FAISS GPU documentation](https://github.com/facebookresearch/faiss/wiki/Faiss-on-the-GPU).

Gains come from parallel distance computation, high memory bandwidth, and processing several queries simultaneously. The authors explicitly warn that small indexes and small batches are often better handled on CPUs. Large indexes and batches are the most favorable GPU scenario. [CPU/GPU comparison](https://github.com/facebookresearch/faiss/wiki/Comparing-GPU-vs-CPU).

Useful implementation entry points:

- [GpuIndexIVFFlat.cu](https://github.com/serenedb/faiss/blob/654e2c8b88484881b6ab79d3e94c32315e46c5ff/faiss/gpu/GpuIndexIVFFlat.cu): GPU-index creation, training, IVF-list copying, and cuVS selection; pinned to the inspected commit of SereneDB's dependency.
- `faiss/gpu/impl/`: search and data-storage implementations.
- `StandardGpuResources`: temporary memory and execution resources.

The last point matters for a server: FAISS GPU is **not considered thread-safe even for reads**, because `StandardGpuResources` manages shared temporary memory. One object cannot simply be handed to every query-execution thread. [FAISS threading rules](https://github.com/facebookresearch/faiss/wiki/Threads-and-asynchronous-calls).

**Lessons for SereneDB:** persistent GPU indexes, reusable resources and buffers, batched search, and explicit resource ownership. Recreating the index and allocating workspace on every SQL query would consume the gains.

## cuVS: specialized GPU algorithms

cuVS is NVIDIA's vector-search and clustering library. It includes IVF-Flat, IVF-PQ, CAGRA, and other implementations. It is used directly and through FAISS. [cuVS source](https://github.com/NVIDIA/cuvs).

Two directions matter here:

- **IVF:** close to the selected thesis topic: select clusters, compute candidate distances, and produce top-K.
- **CAGRA:** a graph algorithm designed specifically for GPUs. It illustrates adapting algorithms and data structures to device architecture.

The CAGRA authors report **2.2–27×** faster construction than HNSW and **33–77×** faster large-batch search at 90–95% recall. These results describe the paper's experiments, not an arbitrary database or SQL workload. [CAGRA, ICDE 2024](https://arxiv.org/abs/2308.15136).

IVF-Flat supports filtering, but within lists already selected for scanning. An eligible document may lie in an unvisited cluster. Filtered recall therefore needs separate measurement while varying probes and filter selectivity. [cuVS IVF-Flat guide](https://docs.nvidia.com/cuvs/user-guide/api-guides/indexing-guide/ivf-flat).

**Lessons:** ready-made GPU primitives and algorithms, device-side filters, and quality/speed curves. Equal `nprobe` alone does not make a comparison between implementations fair.

### Does cuVS improve on classic FAISS GPU?

Meta provides a concrete comparison at **recall@10 = 95%**:

| Data and operation | Classic path | cuVS | Speedup |
|---|---:|---:|---:|
| IVF-Flat search, 100M × 96 | 0.75 ms | 0.39 ms | 1.9× |
| IVF-PQ search, 5M × 1536 | 1.78 ms | 0.22 ms | 8.1× |
| CPU HNSW versus CAGRA construction, 5M × 1536 | 1106.1 s | 89.7 s | 12.3× |

The hardware was NVIDIA H100 and Intel Xeon Platinum 8480CL. **The first two rows compare GPU implementations**, while the last compares different algorithms and devices. They cannot be combined into a claim that “GPUs are eight times faster than CPUs.” [Meta's publication, methodology, and tables](https://engineering.fb.com/2025/05/08/data-infrastructure/accelerating-gpu-indexes-in-faiss-with-nvidia-cuvs/).

## Milvus: GPU search inside a complete vector database

Milvus provides `GPU_CAGRA`, `GPU_IVF_FLAT`, `GPU_IVF_PQ`, and `GPU_BRUTE_FORCE`. In the described GPU mode, indexes are loaded into GPU memory; some variants separately control retention of original vectors for result refinement.

The documentation explicitly notes that a GPU index **may not reduce single-query latency**: maximum throughput requires high load or many query vectors. Operations, metrics, and memory also have constraints. [Milvus GPU index overview](https://milvus.io/docs/gpu_index.md).

This is a useful complete-integration example: collection loading, search parameters, API limits, and memory accounting. An advertised “up to 100×” without a specific batch, index, and recall should not be used as a thesis result.

## Qdrant: GPUs accelerate HNSW construction

Qdrant implements its own GPU HNSW construction through **Vulkan**, supporting different GPU families. This differs from CUDA libraries FAISS/cuVS. The implementation is in [hnsw_index/gpu](https://github.com/qdrant/qdrant/blob/222b2e44ab1c3f17e8246396ead6558e45041df7/lib/segment/src/index/hnsw_index/gpu/mod.rs), separating device management, GPU vector storage, graph builders, and batched point processing.

In a published Qdrant 1.19 example on 100K vectors, construction took about **6.1 s with GPU versus 66.3 s without**. Search throughput stayed approximately equal; another important effect was lower latency spikes because construction consumed less CPU. [Qdrant experiment](https://qdrant.tech/documentation/tutorials-operations/gpu-accelerated-hnsw-indexing/).

**Lessons:** separate preparation speedup from search speedup and measure background work's impact on query p99. This also illustrates benefits without permanent GPU search serving.

## Amazon/OpenSearch: GPUs as a temporary construction service

OpenSearch supports remote vector-index construction:

```text
Database CPU nodes
    → vectors and IDs uploaded to S3
    → GPU service builds the index
    → finished index returns to the database
```

Size thresholds, supported configurations, and flush/merge integration are documented. An open service implementation provides job-submission and status APIs. [OpenSearch architecture](https://docs.opensearch.org/latest/vector-search/remote-index-build/), [Remote Vector Index Builder](https://github.com/opensearch-project/remote-vector-index-builder).

Amazon OpenSearch Service provisions GPU resources on demand. This avoids continuously paying for GPUs needed only during intensive indexing.

A representative result for **10M vectors with 768 dimensions**:

- CPU: 64.1 minutes of indexing and 444.5 minutes of force merge.
- GPU: 21.9 and 14.9 minutes, respectively.
- Total speedup: **13.8×**, while indexing alone improved by **2.9×**.

Most of the total gain comes from merging. This demonstrates why measurement boundaries must be precise. [AWS benchmarks](https://aws.amazon.com/blogs/big-data/build-billion-scale-vector-databases-in-under-an-hour-with-gpu-acceleration-on-amazon-opensearch-service/).

## Oracle: a similar off-database construction model

Oracle AI Database 26ai provides Vector Index Service in Private AI Services Container. `CREATE VECTOR INDEX` can send vectors to a GPU service; the resulting graph returns to Oracle as an HNSW index.

Oracle states that it uses cuVS and claims up to **10× acceleration over a SIMD CPU implementation**. The page lacks enough detail to reproduce the figure independently, so this remains a vendor claim rather than a comparable experiment. The capability established here is **GPU-offloaded index construction**. [Oracle Vector Index Service](https://www.oracle.com/database/private_ai_services_container/vector-indexes/).

## CedarDB: no verified GPU case yet

The inspected CedarDB documentation describes a vector type and **exact nearest-neighbor search**; approximate search is listed as a future direction. No GPU path was established in the checked sources. CedarDB should therefore not be included among implemented GPU systems.

It is useful as an additional CPU baseline for exact search. [CedarDB vector-type documentation](https://cedardb.com/docs/references/datatypes/vector/).

## Other systems and out-of-core search

| System | Research relevance |
|---|---|
| **DuckDB FAISS extension** | Experimental extension supporting named FAISS indexes, insertion, search, and transfer of supported indexes to GPUs. A very close SQL-wrapper example, but not yet integration with SereneDB search indexes. [Code and overview](https://github.com/duckdb-faiss-ext/duckdb-faiss-ext) |
| **Kinetica** | Commercial GPU database combining vector search, filters, and SQL. It describes cuVS use and brute-force handling of fresh data until an index is ready. Internals are closed; vendor comparison figures need separate verification. [Vector-search overview](https://www.kinetica.com/features/vector-search/) |
| **BANG** | Research implementation for datasets larger than VRAM: compressed vectors on GPU, graph in RAM, and overlapping CPU/GPU work. The authors demonstrate gains on billion-scale datasets against the paper's particular baselines. [BANG paper](https://arxiv.org/abs/2401.11324) |
| **FusionANNS** | Cooperative CPU, GPU, and SSD execution: multilevel indexes, filtering and reranking, and reduced repeated I/O. Useful for a later extension beyond fully resident indexes. [FusionANNS paper](https://arxiv.org/abs/2409.16576) |

## Detailed candidates

| Candidate | What is accelerated and demonstrated | Thesis relevance |
|---|---|---|
| **pg_cuvs** | Actual SQL search through resident cuVS and a GPU daemon. Raw results and kernel/IPC/SQL timing breakdowns are published. | A close task: the GPU returns candidates, the database fetches rows and checks visibility. [Detailed analysis](gpu-candidates/pg-cuvs.md) |
| **VecFlow** | GPU filtered ANN: label-specific graphs for long lists and brute-force for short lists. | Algorithm choice based on the eligible subset, ID mapping, and filtered benchmarks; discussed below. [Analysis](gpu-candidates/vecflow.md) |
| **PGPU / VectorChord** | GPU-computed IVF centroids, followed by VectorChord serving. The inspected README did not establish a complete numerical CPU/GPU benchmark. | A small integration boundary for faster construction; not a substitute for GPU search. [Analysis](gpu-candidates/pgpu.md) |
| **LanceDB / Lance** | Published IVF4096 training on 1M × 1536: L4, 323 → 12.5 s; M2 Max, 397 → 21 s. | Accelerates k-means, not the whole query or build. Current code also has other GPU build stages. [Analysis](gpu-candidates/lancedb.md), [authors' experiment](https://www.lancedb.com/blog/gpu-accelerated-indexing-in-lancedb-27558fa7eee5) |
| **RUMMY** | IVF search with lists fetched from RAM to one A100 40GB; reported 2.4–23.1× over IVF-CPU on 64 vCPUs, batches 8/2048. | Fewer repeated cluster transfers and transfer/compute overlap; an extension after the resident MVP. [Analysis](gpu-candidates/rummy.md), [NSDI 2024 paper](https://www.usenix.org/system/files/nsdi24-zhang-zili-pipelining.pdf) |
| **GGNN** | Independent CUDA graph construction, search, and sharding. A comparative study includes CAGRA and other GPU graphs. | Alternative algorithms and batch effects; not a pure CPU/GPU IVF comparison. [Analysis](gpu-candidates/ggnn.md) |
| **Jasper** | A 2026 study of GPU Vamana, batch updates, and RaBitQ; paper and code are available. | Derived-structure invalidation contracts, peak workspace, and profiling. Graphs and quantization exceed the MVP. [Analysis](gpu-candidates/jasper.md) |
| **Elasticsearch GPU indexing** | Builds CAGRA on GPU and exports HNSW. On 138 million 1,024-dimensional vectors, eight GPUs made indexing 7× faster. During indexing, CPU search p90 latency dropped from 45 ms to 7 ms. | Compatible storage format and isolating CPUs from build load. [Analysis](gpu-candidates/elasticsearch.md), [Elastic benchmark](https://www.elastic.co/search-labs/blog/vector-indexing-gpu-elasticsearch-cuvs) |

## VecFlow: filtered ANN and a different meaning of IVF

VecFlow builds label-centric posting lists: graphs for long label lists and GPU brute-force for short ones. Here IVF means grouping by attributes, unlike the thesis's centroid-based IVF-Flat. Useful ideas include selecting an algorithm by eligible-candidate count, a shared vector array with ID mapping, and a scan-oriented layout. [Authors' architecture](https://supercomputing-system-ai-lab.github.io/projects/vecflow/filtered/), [detailed analysis](gpu-candidates/vecflow.md).

The paper reports 5M QPS at recall 0.90 on labeled SIFT1M and up to 135× over StitchedVamana. This is a filtered-search experiment, not ordinary SQL. The authors' arXiv preprint was inspected; agreement with the directly inaccessible ACM PDF was not verified. [Preprint v1](https://arxiv.org/html/2506.00812v1).

The inspected C++ example holds query vectors and labels on the device before timing. It measures the entire batch with synchronization, excluding initial upload and returning answers to the client; `avg` is not single-query latency. Keep the example, paper, and future SQL-backend figures separate. [Timer](https://github.com/Supercomputing-System-AI-Lab/VecFlow/blob/b5d1cbe433e3b632d5e5758209fa2e5f6f73cd7d/vecflow/examples/cpp/src/vecflow_example.cu).

The smallest useful thesis extension is investigating GPU scans of eligible subsets, including preparation cost. The full label-centric architecture, graphs, and persistent queue exceed the MVP. VecFlow-Chamfer in the same repository belongs to separate multi-vector work; its results must not be mixed with single-vector filtered ANN. [Scope and API](gpu-candidates/vecflow.md).

## Benchmark qualifications

Elasticsearch publishes its Rally configuration: eight GPUs/data nodes, `vectors-only`, replicas=0, and disabled refresh during bulk loading. This is not complete indexing of every field with ordinary visibility-refresh behavior. [Experimental configuration](https://github.com/elastic/rally-tracks/issues/1188).

DuckDB FAISS extension's GPU function is verified in code, but the 2024 thesis cited in its README measures CPU paths. Its VSS comparison cannot be cited as GPU acceleration. Verification of the full GPU search-parameter set also needs attention. [Inspected adapter and evidence limitations](gpu-candidates/duckdb-faiss.md).

Pin the Jasper paper version: v5 dated July 31, 2026 differs from early v1 in results and conditions. Peak QPS without recall may describe an unusable operating point. Blanket claims that CAGRA cannot be updated also need qualification: modern cuVS has `extend`. [Jasper paper v5](https://arxiv.org/html/2601.07048v5), [CAGRA API](https://docs.nvidia.com/cuvs/user-guide/api-guides/indexing-guide/cagra).

## Benchmark selection

| Tool | What to reuse | Required adaptation |
|---|---|---|
| **cuVS Bench** | FAISS/cuVS/CPU comparisons, sweeps, Pareto curves, and separate build/search timers | IResearch/SereneDB backend; verification of measurement boundaries. [Documentation](https://docs.nvidia.com/cuvs/user-guide/benchmarking-guide/cu-vs-bench-tool/usage) |
| **VIBE** | Modern embeddings and GPU algorithms | An index benchmark; published CPU results are single-core. [Project](https://github.com/vector-index-bench/vibe) |
| **VectorDBBench** | Server latency, throughput, concurrency, and filters | SereneDB client and actual CPU/GPU-path verification. [Project](https://github.com/zilliztech/VectorDBBench) |
| **Big ANN Benchmarks** | Data, ground truth, and filtered/OOD/streaming scenarios | Restrict scale and scenarios to thesis scope. [Project](https://github.com/harsha-simhadri/big-ann-benchmarks) |
| **ANN-Benchmarks** | Classic small datasets and historical baselines | Active maintenance has stopped; the authors recommend VIBE for new infrastructure. [Status](https://github.com/erikbern/ann-benchmarks) |

Recommendation: start with SIFT1M for validation and a real text dataset such as VIBE arxiv-nomic for the main analysis, then scale within VRAM capacity. Base subsets and filtered selections need corresponding exact ground truth. Measure batch throughput separately from complete server latency. Data, metric definitions, limitations, and the proposed protocol are collected in [gpu-candidates/benchmarks.md](gpu-candidates/benchmarks.md).
