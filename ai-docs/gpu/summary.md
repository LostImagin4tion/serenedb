The research identifies four main uses of GPUs in databases:

- **Vector search:** FAISS GPU, cuVS, Milvus and VecFlow execute distance calculations and top-K selection on the GPU.
- **Index construction:** Qdrant, Elasticsearch, OpenSearch and Oracle build indexes on GPUs while keeping query execution on CPUs.
- **SQL execution:** Velox-cuDF, Sirius, HeavyDB and PG-Strom offload operators or larger analytical pipelines.
- **External computation:** ClickHouse and Snowflake invoke GPU-backed models or services.

The gains come from parallelism, memory bandwidth, reusing data in VRAM, reducing intermediate transfers, and overlapping transfers with computation. Offloading construction also frees CPU resources for concurrent queries.

Some representative published results:

| Case | Result | Conditions |
|---|---|---|
| [LanceDB IVF training](https://www.lancedb.com/blog/gpu-accelerated-indexing-in-lancedb-27558fa7eee5) | **323 → 12.5 s** | IVF4096, L2, 1M × 1536, NVIDIA L4; k-means training only, not the complete build |
| [FAISS IVF-Flat](https://engineering.fb.com/2025/05/08/data-infrastructure/accelerating-gpu-indexes-in-faiss-with-nvidia-cuvs/) | **0.75 → 0.39 ms** | Classic GPU versus cuVS GPU; H100, 100M × 96, recall@10 = 0.95, single-query calls |
| [Elasticsearch](https://www.elastic.co/search-labs/blog/vector-indexing-gpu-elasticsearch-cuvs) | **7× indexing throughput**; concurrent CPU-search p90 **45 → 7 ms** | 138M × 1024, eight GPUs; search remains on CPU |
| [RUMMY](https://xinjin.github.io/files/NSDI24_Rummy.pdf) | **2.4–23.1× over IVF-CPU** | One A100 versus 64 vCPUs; data exceeds VRAM; batches of 8/2048 |
| [Velox-cuDF / Presto](https://developer.nvidia.com/blog/accelerating-large-scale-data-analytics-with-gpu-native-velox-and-nvidia-cudf/) | **1246 → 133.8 s**, approximately **9.3×** | SF1000, 21 TPC-H-derived queries; AMD 7965WX versus RTX PRO 6000 Blackwell; CPU Q21 did not complete |
| [gpudb](https://github.com/singhpratech/duckdbgpumetaldbram) | **32× on-device, 1.5× end-to-end** | GROUP BY returning 75M groups; initial upload and sort excluded |

These results measure different operations and hardware configurations. They illustrate mechanisms rather than establish a ranking. The gpudb example particularly shows why kernel speedup must be separated from complete-query speedup.

Available benchmarks and public results cover several levels:

- **[cuVS Bench](https://docs.nvidia.com/cuvs/user-guide/benchmarking-guide/cu-vs-bench-tool/usage):** standalone FAISS/cuVS/CPU comparisons and parameter sweeps.
- **[VIBE](https://vector-index-bench.github.io/) and [Big ANN](https://github.com/harsha-simhadri/big-ann-benchmarks):** datasets and recall/throughput comparisons, including modern embeddings and filtered/OOD scenarios.
- **[VectorDBBench](https://zilliz.com/vdbbench-leaderboard):** database-level QPS, latency, recall, filters and concurrent ingestion.
- **TPC-H / [ClickBench](https://benchmark.clickhouse.com/):** analytical SQL workloads and published cross-system results.
- **Elastic Rally:** indexing, merging and search under background load.

For SereneDB, I would start with **profiling the existing IVF build, then a bounded GPU-assisted construction experiment**. Offload centroid training and, if justified by the profile, vector-to-cluster assignment; return the results to the existing IResearch writer and retain CPU search. LanceDB and PGPU provide relevant examples of this boundary.

Measure complete build time, including setup and transfers, and verify the resulting index's search recall. The baseline must use SereneDB's current optimized CPU builder, including SuperKMeans where selected. A large training speedup does not guarantee a faster complete build: other stages and small training tasks within the centroid tree can dominate.

**Resident IVF-Flat through FAISS GPU is the subsequent query-integration stage**, retaining one GPU, FP32 vectors and an immutable snapshot. It remains the agreed thesis deliverable; GPU-assisted construction alone does not complete it. This stage adds the data/ID adapter, GPU-copy lifecycle and actual GPU-backed server queries.

The GPU-query comparison should include the existing CPU server path and standalone FAISS CPU/GPU baselines at matched recall. Since SereneDB defaults to SQ8 for L2/IP/cosine, use an explicit `quant = 'none'` CPU baseline for the uncompressed comparison.

Possible extensions after the query MVP are filtered GPU scans and out-of-core IVF inspired by RUMMY. Broader SQL offload would be a separate, substantially larger project.

For the construction experiment, record stage timings, complete build time, resulting recall and peak RAM/VRAM. For GPU serving, measure preparation/loading, warm-query p50/p95/p99 and throughput separately, with batch size and client concurrency recorded explicitly.
