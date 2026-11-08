# Candidates for a SereneDB GPU vector-search backend

This extends the [general survey](../README.md), selecting systems against the [agreed thesis requirements](../../serenedb_gpu_diploma.md): FAISS GPU, uncompressed IVF, FP32, one GPU, a resident immutable snapshot, and a complete server query.

Here, a “candidate” is a source of architectural ideas or a comparison implementation. Including a graph index does not change the thesis's selected IVF backend. These systems were not built locally, and their results were not reproduced. Figures come from the cited authors; implementation effort and proposed experiments are our assessments.

## Selected systems

| Candidate | GPU role | Why read it | Thesis priority |
|---|---|---|---|
| [FAISS GPU](faiss-gpu.md) | Flat/IVF training, insertion, and search | Main library; resources, formats, and adapter interface | Primary |
| [LanceDB / Lance](lancedb.md) | IVF/PQ training and cluster assignment | Separate index construction from serving; CUDA and MPS | Primary for the initial construction experiment |
| [PGPU / VectorChord](pgpu.md) | Clustering for IVF construction | A small GPU-integration boundary inside a SQL database | Primary for the initial construction experiment |
| [cuVS](cuvs.md) | IVF, graph-based CAGRA, exact search | Compare IVF implementations and obtain a ready-made benchmark harness | Primary |
| [DuckDB FAISS extension](duckdb-faiss.md) | Search through supported FAISS GPU indexes | A close example of a SQL wrapper and its limitations | Primary |
| [pg_cuvs](pg-cuvs.md) | Resident search; separate GPU-build/CPU-search mode | Integration costs, row IDs, IPC, visibility, and diagnostics | Primary, as an experimental example |
| [Milvus](milvus.md) | GPU build and search; possible adaptation to CPU | GPU-index lifecycle in a database server | Primary |
| [VecFlow](vecflow.md) | Filtered ANN: graphs for frequent labels, brute-force for rare labels | Label-centric indexing, filtered ground truth, and eligible-subset scan costs | Secondary; primary when adding filters |
| [RUMMY](rummy.md) | IVF search with lists fetched from RAM | Scheduling transfers and computation for data larger than VRAM | Secondary; extension after the MVP |
| [Elasticsearch GPU indexing](elasticsearch.md) | CAGRA construction with HNSW export | Storage compatibility, merges, ingestion speed, and search impact | Secondary |
| [Qdrant](qdrant.md) | HNSW construction through Vulkan | An alternative GPU stack and indexing/search contention | Secondary |
| [GGNN](ggnn.md) | Graph search and construction, sharding | An independent GPU baseline and batch-size effects | Supplementary |
| [Jasper](jasper.md) | Vamana, updates, and RaBitQ | A 2026 study: memory, updates, and profiling | Supplementary |

New detailed directions relative to the previous survey: **pg_cuvs, PGPU, LanceDB, RUMMY, GGNN, Jasper, and Elasticsearch**. The other entries have deeper source-code checks and benchmark analysis.

## How to interpret the results

Each document separates the GPU's role, data movement, published results, limitations, implementation effort, and a useful experiment. Links to inspected files are pinned to commits. Unpinned documentation pages reflect their state when checked; paper versions are identified separately.

Keep these comparisons separate:

- build speedup and search-query speedup;
- GPU versus CPU and a new GPU implementation versus an older one;
- kernel time, library-call time, and SQL time including the client round trip;
- large-batch QPS and single-query latency;
- results with different recall, vector types, or device counts.

The [benchmark document](benchmarks.md) covers construction measurements first, followed by library and server search. Start with the existing SereneDB CPU builder and a bounded GPU-training experiment. For the query stage, use **cuVS Bench for the library, VectorDBBench or a small client adapter for the server, and VIBE and Big ANN datasets**. These are proposals for future experiments; the tools and data were not installed.

## Why not give every discovered system a separate document?

For the main implementation, proximity to IVF and verifiable integration matter more than the number of systems listed. BANG and FusionANNS remain related work for datasets larger than VRAM: [BANG](https://arxiv.org/abs/2401.11324), [FusionANNS](https://arxiv.org/abs/2409.16576). SONG, CUHNSW, and GANNS can be studied through the [HardBD & Active 2025 comparison](https://www.shimin-chen.com/papers/gpu-graph-anns-hardbdactive25.pdf), included in the GGNN and benchmark notes.

AWS/OpenSearch and Oracle are covered in the [vector-search survey](../gpu-vector-search.md). Elasticsearch provides additional coverage of their GPU-build/CPU-search model. ClickHouse and Snowflake are covered in [GPU-backed services](../gpu-services.md); they were not selected for deeper ANN analysis because their verified scenarios include external computation and inference, which are further from resident IVF. CedarDB was not selected because the checked sources did not establish a GPU implementation.

## Recommendation from the selection

Begin with the [construction experiment](../gpu-serenedb.md#initial-experiment-gpu-assisted-ivf-construction): profile the current CPU build and investigate GPU training/assignment, using LanceDB and PGPU as the closest references. Judge the complete build and resulting recall, rather than training speed alone.

Then keep FAISS GPU for the agreed resident-query MVP and separately compare its cuVS variant. The main server lessons concern resource ownership, GPU-copy versions, and measurement boundaries. Construction acceleration is an initial experiment, while GPU-backed server queries remain the thesis deliverable. RUMMY, graphs, updates, and VecFlow-style filtering remain possible extensions after that MVP.
