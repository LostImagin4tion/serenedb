# GPU vector-search benchmarks and their use in SereneDB

Base snapshot: **October 28, 2026**; VecFlow addition: **October 31, 2026**; public-results addition: **November 7, 2026**. This is a tool survey and proposed protocol for future experiments. Harnesses were not installed, datasets were not downloaded, and no local CPU/GPU measurements were performed.

## What to choose

Start with a profile of the existing SereneDB IVF build and a bounded GPU-assisted construction experiment. Then evaluate the two query levels: library ANN search and actual SereneDB queries. A practical choice for that query stage is **cuVS Bench for FAISS/cuVS and a server adapter for SereneDB**, using the same data and ground truth. VIBE supplies modern embeddings and independent methodology; VectorDBBench supplies a database-load framework.

| Tool | Level | Reusable elements | Limitation |
|---|---|---|---|
| cuVS Bench | Libraries; extensible backends | FAISS/cuVS/CPU baselines, sweeps, build/search, Pareto curves | Verify timer boundaries; no ready-made SereneDB backend was established |
| VIBE | ANN algorithms | Modern embeddings, OOD datasets, CPU/GPU wrappers | Not intended to compare complete databases |
| VectorDBBench | Database servers | Load generation, recall, latency, concurrency, filters | Needs a SereneDB adapter and actual-backend verification |
| Big ANN Benchmarks | Algorithms and specialized ANN scenarios | Large datasets, formats, and filtered/OOD/streaming tracks | Scale and some tracks exceed the MVP |
| ANN-Benchmarks | Classic ANN libraries | Small datasets, ground truth, and older baselines | No longer actively maintained by its authors |
| EnterpriseDB vsbt | PostgreSQL extensions | Real data, SQL load/query, warmup, and client load | PostgreSQL-specific; not a ready-made SereneDB runner |
| Elastic Rally | Elasticsearch | Ingest/search/merge and concurrent-load examples | Useful methodology; not a universal SQL runner |
| VecFlow examples | Filtered ANN | Labeled data, filtered ground truth, and parameter sweeps | Specialized library runner; device-resident batch time is not SQL latency |

## Public results and leaderboards

These sites publish existing results, rather than only benchmark runners. No single portal covering every system in this survey was found. Checked on **November 7, 2026**.

| Site | Coverage and available results | Interpretation |
|---|---|---|
| [VectorDBBench Leaderboard](https://zilliz.com/vdbbench-leaderboard) | Milvus, Qdrant, Elasticsearch, OpenSearch, and others; QPS, latency, recall, filters, and search during ingestion | Published by Zilliz; preserve the exact configuration and compare at matched quality |
| [Qdrant Benchmarks](https://qdrant.tech/benchmarks/) | Database comparisons with RPS, latency, p95, indexing time, and [downloadable raw results](https://qdrant.tech/benchmarks/results-1-100-thread-2024-06-15.json) | Published by Qdrant; the displayed main single-node run is from January/June 2024 |
| [ClickBench](https://benchmark.clickhouse.com/) | SQL analytics, including ClickHouse, DuckDB, and HeavyAI; query timings, load time, and CPU/GPU filtering | Different workload from ANN; inspect the selected machine, system version, and run settings. [Results and reproduction repository](https://github.com/ClickHouse/ClickBench) |
| [VIBE results](https://vector-index-bench.github.io/) | Interactive ANN algorithm results, including CPU/GPU methods; recall/throughput curves and configurations | Index-level measurements, rather than complete database-server queries |

### Example published database results

The VectorDBBench page's **$1,000 monthly cost** comparison reports these points for **Cohere 1M × 768 dense float vectors**. Configuration labels are retained verbatim; QPS values are published results, not local measurements. [Source table](https://zilliz.com/vdbbench-leaderboard).

| Published configuration | QPS | Recall |
|---|---:|---:|
| `Milvus-16c64g-sq8-force_merge` | 5,973 | 0.9192 |
| `OpenSearch-16c128g-force_merge` | 3,055.01 | 0.9066 |
| `ElasticCloud-8c60g-force_merge` | 2,353.89 | 0.9143 |
| `QdrantCloud-16c64g` | 1,242.43 | 0.9474 |

These points differ in configuration and achieved recall. Their QPS ratios do not establish speedup at equal quality. Use recall/performance curves to select a common quality target, and verify the execution device in each run's configuration: a database leaderboard entry alone does not establish GPU execution.

For public results, use VectorDBBench for server comparisons, VIBE for ANN algorithms, and ClickBench for analytical SQL. Keep those measurement levels separate when selecting SereneDB baselines.

## Initial construction experiment

Use the existing SereneDB CPU builder as the end-to-end baseline. Profile it before selecting GPU work; the first candidate is centroid training, with assignment added if the profile justifies it. See the [integration rationale](../gpu-serenedb.md#initial-experiment-gpu-assisted-ivf-construction).

1. Fix the base/query data and exact ground truth. Record the training sample, seed, metric, normalization, precision, cluster/tree settings, iterations, quantizer and CPU threads.
2. Record reading/sampling, centroid training, assignment, quantization/encoding and writing. For GPU work, include context/resource setup, allocation, upload, execution and download. Also measure complete build wall time; overlapping stage timers must not simply be added.
3. Compare with the actual optimized CPU selection, including SuperKMeans where used. If the GPU experiment changes the clustering algorithm, sampling or training budget, identify that change explicitly.
4. Keep downstream storage and CPU query serving equivalent. Measure recall and search latency across query settings for both resulting indexes; training inertia alone is insufficient.
5. Save peak RAM/VRAM, actual GPU stages, raw timings and failures. Record the distribution of training-task sizes in the centroid hierarchy so launch/transfer overhead can be explained.

The construction experiment is complete with a reproducible profile, a bounded GPU-offload comparison, and an explanation of build-time and quality effects. A speedup is not assumed. The subsequent query protocol below still covers the agreed server-side GPU-search result.

## cuVS Bench

The main advantage is a common tool for different ANN libraries. It separates build/search and sweeps configurations. Reports distinguish GPU time, wall-clock batch latency, recall, queries per batch, and total throughput. `end_to_end` describes the benchmarked operation: it does not promise to include index loading and SQL. [Metric definitions](https://docs.nvidia.com/cuvs/user-guide/benchmarking-guide/cu-vs-bench-tool/usage).

The pinned interface allows a `ConfigLoader` and `BenchmarkBackend` with `build()`/`search()` returning `BuildResult`/`SearchResult`. This supports integrating a server or in-process SereneDB path. Following the contract matters more than inventing an incompatible result format. [Backend API, commit `281331f`](https://github.com/NVIDIA/cuvs/blob/281331f736886ced474459f552c76822cde03d09/fern/pages/cuvs_bench/pluggable_backend.md).

**Query-stage proposal:** after the construction experiment, run the existing FAISS CPU/GPU search baselines without SereneDB. Then choose between adding a cuVS Bench backend and saving compatible data/results from a small server runner. This concerns future harness implementation; no new adapter has been written.

## VIBE

VIBE, Vector Index Benchmark for Embeddings, is a 2026 paper and toolset. It targets modern embeddings, in-distribution/OOD scenarios, vector representations, and GPU algorithms. Adding a method requires a wrapper, container, and parameter grid. [Project](https://github.com/vector-index-bench/vibe), [pinned README](https://github.com/vector-index-bench/vibe/blob/7ac76c1f8320a411ce7c9cb3791bdd7f2349b30c/README.md).

In the results available when checked, CPU algorithms use **one Xeon Gold 6230 core**, while GPU algorithms use V100 32GB. Their CPU/GPU ratio therefore cannot be used directly as a win over tuned multithreaded SereneDB. A separate multicore baseline is necessary. [VIBE conditions](https://github.com/vector-index-bench/vibe).

The `arxiv-nomic-768-normalized` dataset is particularly useful: 1,344,643 vectors. It is closer to text retrieval than SIFT alone. Normalized vectors allow aligned cosine/IP/L2 rankings, but numerical distances and API parameters must still match the selected metric.

## VectorDBBench

This measures a database server rather than just a library. It includes clients for different systems, filtered search, and concurrency; SereneDB needs its own client adapter. [Repository and code](https://github.com/zilliztech/VectorDBBench/tree/700a03b9cef504b6ba86a2ff928c8ef1e48ce424).

An important detail in the current report: for NQ > 1, QPS counts **query vectors**, while batch latency covers the entire request. Serial latency/recall use NQ=1. Not every backend supports NQ > 1. [Definitions and limitations](https://github.com/zilliztech/VectorDBBench).

The tool comes from Milvus/Zilliz's developer; that does not invalidate the code, but independent configurations, raw results, and selection rules matter more than the leaderboard. The thesis should use recall/QPS/latency curves rather than a combined score.

## Big ANN and ANN-Benchmarks

Big ANN provides a framework for large datasets and practical search variants. The 2023 competition separates filtered, OOD, sparse, and streaming tracks; for example, filtered uses YFCC10M and OOD uses Text-to-Image10M. [Framework](https://github.com/harsha-simhadri/big-ann-benchmarks), [competition and dataset description](https://papers.neurips.cc/paper_files/paper/2025/file/63092d79154adebd7305dfd498cbff70-Paper-Datasets_and_Benchmarks_Track.pdf).

The MVP does not need a billion vectors. Reuse formats, existing ground truth, and subtasks. Filtered/OOD are possible extensions: they expose limitations hidden by ordinary in-distribution ANN.

ANN-Benchmarks provides HDF5 datasets with base/query/ground truth, but its README explicitly states that active maintenance has stopped and points to VIBE. Its data and historical results remain usable; building new infrastructure around an unmaintained runner needs justification. Its published CPU results are also not a multicore comparison with GPUs. [Status and datasets](https://github.com/erikbern/ann-benchmarks).

## Two additional methodology sources

**EnterpriseDB vsbt:** useful features include separate load/build/query phases, PGPU configurations, parallel SQL clients, and explicit warmup control. Its README distinguishes subsets from original datasets and discusses recomputing ground truth. Some orchestration scenarios change PostgreSQL settings and restart it, so they must not be treated as harmless query runners. [Code and workflow](https://github.com/EnterpriseDB/vsbt/tree/65e3a4e98968e1d5040b79dc3c08c96216394338).

**Rally:** an example of separately measuring indexing, force merge, and search under background ingestion. Elastic's published GPU configuration uses eight shards/GPUs, vectors-only data, and disabled refresh during bulk loading. Keep those conditions alongside the result. [Experimental settings](https://github.com/elastic/rally-tracks/issues/1188).

## Datasets for the first stage

This proposed small selection covers different dimensions and data characteristics. Sizes below are **calculated as `N × d × 4`** for FP32 vectors only, excluding indexes, IDs, and workspace; GB is decimal. These are not measured VRAM requirements.

| Data | Shape | FP32 vectors | Purpose |
|---|---:|---:|---|
| SIFT1M | 1M × 128 | 0.512GB | Correctness and comparison with classic studies |
| GIST1M | 1M × 960 | 3.84GB | High dimensionality; small query set |
| VIBE arxiv-nomic | 1,344,643 × 768 | ≈4.13GB | Real text embeddings |
| Deep10M | 10M × 96 | 3.84GB | Increasing N at low dimensionality |
| OpenAI embeddings | 5M × 1536 | 30.72GB | VRAM pressure; only with sufficient capacity |

Sources and exact variants: [SIFT/GIST](https://github.com/erikbern/ann-benchmarks), [VIBE](https://github.com/vector-index-bench/vibe), [cuVS dataset guide](https://docs.nvidia.com/cuvs/user-guide/benchmarking-guide/cu-vs-bench-tool/datasets), [OpenAI dataset in Meta's comparison](https://engineering.fb.com/2025/05/08/data-infrastructure/accelerating-gpu-indexes-in-faiss-with-nvidia-cuvs/).

**Keep variants of the same name separate.** Different studies use Deep at 10M/100M/1B, L2 or angular distance, and different preprocessing. SIFT may be stored as uint8, while the thesis backend requires FP32. Record the source URL, revision/hash, dtype, conversion, and metric in the manifest.

Reducing the base to a subset usually invalidates the old ground truth: nearest neighbors must be determined among the remaining data. The same applies to SQL filters. Removing absent IDs from the old top-100 does not produce an exact new top-K.

## October 31 addition: VecFlow's contribution to filtered benchmarks

This addition concerns a possible extension after the MVP works. It does not require immediately adding complex SQL filters to the thesis GPU backend. [VecFlow analysis and primary sources](vecflow.md).

**Keep three levels separate.** The inspected paper is arXiv v1 from 2025; the current C++ example is pinned 2026 code; a future SereneDB SQL runner is a third experiment. A paper's multiplier cannot be transferred to the example or SQL without measurement.

The C++ example uploads queries and labels to the GPU in advance; timing covers repeated `search + sync_stream` over the whole batch. `avg` is batch latency; QPS counts query vectors. Initial upload, construction, and returning answers to the client are excluded. [Inspected timer](https://github.com/Supercomputing-System-AI-Lab/VecFlow/blob/b5d1cbe433e3b632d5e5758209fa2e5f6f73cd7d/vecflow/examples/cpp/src/vecflow_example.cu).

The two-label AND generator imposes a minimum intersection size; it leaves `-1` when no suitable pair exists, and the example skips those rows. This is useful for reproducing the example but does not cover empty and very small intersections. [Generator and seed](https://github.com/Supercomputing-System-AI-Lab/VecFlow/blob/b5d1cbe433e3b632d5e5758209fa2e5f6f73cd7d/vecflow/examples/python/generate_multi_query.py), [query handling](https://github.com/Supercomputing-System-AI-Lab/VecFlow/blob/b5d1cbe433e3b632d5e5758209fa2e5f6f73cd7d/vecflow/examples/cpp/src/vecflow_example.cu).

The proposed future SereneDB experiment should preserve these independent axes:

| Axis | What to vary and record |
|---|---|
| Eligible-subset size | `s = eligible/N`; separate rare- and frequent-label groups, not just the mean |
| Document labels | Labels/vector; duplication and mapping costs in RAM/VRAM |
| AND | Intersection size, empty results, `<K`, small and large subsets |
| Relation to geometry | Independent random labels and labels correlated with neighborhoods |
| Workload | Batch=1 and batch throughput separately; query-label frequency distribution |
| Integration cost | SQL predicate/bitmap, vector retrieval, transfers, search, and row return |

Start with existing SIFT1M plus reproducible labels; use filtered YFCC and Wiki-ANN from the [VecFlow analysis](vecflow.md) for more realistic data. These are inputs for testing hypotheses, not one universal fixed workload. A changed distribution or subset requires corresponding exact ground truth.

A minimal SereneDB ablation compares exact CPU scanning of the eligible subset, the same GPU scan with resident data, and the complete GPU path with preparation/transfers. Then, if correct filtered ANN is supported, add it at the same target recall. This separates device gains, the effect of fewer candidates, and SQL-integration costs.

When testing persistent kernels, keep client count, client load, and quality identical between ordinary and persistent modes. Record queue wait, p95/p99, reserved GPU resources, and competing work separately. The repository's persistent CAGRA example uses different thread counts for its two variants; it is not a clean kernel-only ablation. [Example code](https://github.com/Supercomputing-System-AI-Lab/VecFlow/blob/b5d1cbe433e3b632d5e5758209fa2e5f6f73cd7d/examples/cpp/src/cagra_persistent_example.cu).

## Metrics and definitions

For ANN correctness, use the mean:

```text
Recall@K = mean_q |ANN_K(q) ∩ Exact_K(q)| / K
```

Distinguish `1-recall@K`: whether the **first** exact neighbor occurs among the K returned results. This differs from overlap with the K nearest neighbors. Define an ID tie-breaking rule or equal-distance acceptance rule in advance. For filters, compute ground truth within the eligible subset; if fewer than K documents are available, use the available count as the denominator and handle empty subsets as a separate case.

| Metric | Measurement boundary |
|---|---|
| GPU kernel time | CUDA events/device profiler; not SQL latency |
| Library wall time | Complete synchronized call with specified input/output placement |
| Server latency | From client request submission to answer receipt |
| Batch latency | Completion of the whole batch, not time for one row |
| Vector throughput | Processed query vectors / wall time |
| Request throughput | Completed requests / wall time |
| Build/load | Training, construction, serialization, and upload separately |
| Memory | Steady and peak RAM/VRAM, including temporary resources |

`batch_time / batch_size` is amortized cost per vector. It is not an individual request's user-visible latency. QPS also does not always equal `1 / p50`, particularly under concurrency.

## Proposed SereneDB protocol

This is the GPU-query protocol following the initial construction experiment.

1. Fix the data, one metric, and an immutable snapshot. Confirm exact ground truth and ID mapping on a small dataset. Exclude embedding generation from the ANN timer.
2. Obtain the existing SereneDB CPU baseline. Measure FAISS CPU/GPU separately to understand the library boundary; do not substitute them for the main comparison.
3. Sweep each algorithm's search parameters and select points at recall of at least 0.90/0.95/0.99. Explicitly exclude settings that miss the threshold. Prefer a Pareto plot to one speedup number.
4. For IVF, separately investigate identical centroids/lists and independently tuned CPU/GPU indexes. The first better isolates device/implementation effects; the second captures practically achievable performance.
5. Measure cold prepare/load and warm search. Batch=1 is required at both library and server levels; batch=8/32/128/1024 can be explored in a standalone prototype where feasible. A special server batch API is not a thesis requirement.
6. Vary server concurrency separately: for example, 1/4/16/64 clients, recording available CPU threads. Save throughput, latency, and errors at each point. This is a proposed grid, not a promise to run every combination.
7. Repeat the main points at least three times after identical warmup. Save every run, not just the best. p99 requires enough observations; label repeated short query sets as cache-hot reuse.
8. Record packing, H2D/D2H, queue wait, kernels, and document retrieval separately. Asynchronous stages can overlap: their timings cannot automatically be added. Profile in a separate run and check profiling overhead.
9. Verify the selected backend through diagnostics. Hidden CPU fallback must not count, and OOM/timeouts must not be silently skipped. Failed points remain in the results.

This protocol is a recommendation, not a change to the agreed thesis scope. The main result remains one correct GPU IVF path and an explainable comparison.

## Comparison pitfalls found in the sources

| Risk | Example | Thesis response |
|---|---|---|
| Low-recall QPS looks record-breaking | Jasper quotes a peak near recall 0.23 | Always show quality beside the figure |
| GPU-kernel speedup is presented as query speedup | pg_cuvs separately exposes daemon/SQL overhead | Separate timers and add a raw-library baseline |
| Large-batch time is called single-query latency | GGNN and other GPU studies examine batch throughput | Save batch latency, NQ, and QPS separately |
| k-means acceleration is presented as build acceleration | LanceDB's IVF-training benchmark | Measure every build stage |
| Memory migration is compared with a resident index | RUMMY includes a unified-memory baseline | Specify residency mode and transfer volume |
| GPU construction improves search indirectly | Qdrant and Elasticsearch | Identify the resource-contention effect |
| Filtered-query selection changes the workload | VecFlow's example restricts AND-intersection size and skips unsuitable queries | Preserve threshold, seed, intersection distribution, and skipped count |

Detailed sources and figures appear in [Jasper](jasper.md), [pg_cuvs](pg-cuvs.md), [GGNN](ggnn.md), [LanceDB](lancedb.md), [RUMMY](rummy.md), [Qdrant](qdrant.md), [Elasticsearch](elasticsearch.md), and [VecFlow](vecflow.md).

## What to save with every result

Minimum experiment record: SereneDB and library commits; build type and flags; CUDA/driver; GPU model/count, VRAM, and MIG; CPU, threads, NUMA, and RAM; dataset/hash/metric/dtype; build/search settings and seed; NQ/concurrency/load model; warmup; data placement and included transfers; query/repetition counts; recall, latency distribution, throughput, errors, peak memory, and raw results.

A closed-loop server test establishes sustainable load at a fixed client count. For latency at a specified arrival rate, a separate open-loop scenario that accounts for queuing and overdue queries is useful. No ready-made leaderboard replaces this description of conditions.
