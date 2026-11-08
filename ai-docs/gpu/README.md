# GPU research

GPUs are used for index construction, search-query processing, and SQL execution. These tasks have different economics, integration boundaries, and constraints.

Speedup figures come from authors' and vendors' publications; no local measurements were performed for this survey.

## Documents

| Document | Scope |
|---|---|
| [High-level summary](summary.md) | GPU use cases, performance mechanisms, representative numerical results, and implementation directions for SereneDB |
| [Vector search and index construction](gpu-vector-search.md) | FAISS/cuVS, resident and out-of-core ANN, filtered search, GPU-built indexes, and published results |
| [SQL offload and analytical processing](gpu-sql-offload.md) | Velox, Sirius, HeavyDB, PG-Strom, and other query engines; operator offload, full-plan execution, and SQL benchmarks |
| [GPU-backed services](gpu-services.md) | ClickHouse embeddings and Snowflake GPU compute pools; model/service execution boundaries |
| [Applying the research to SereneDB](gpu-serenedb.md) | Performance mechanisms, integration recommendations, comparison levels, and preparation-cost amortization |
| [Candidate profiles](gpu-candidates/README.md) | 13 detailed documents with source-code checks, benchmark conditions, and implementation-effort estimates |
| [Benchmarks and public results](gpu-candidates/benchmarks.md) | Harnesses, leaderboards, datasets, metric definitions, and the proposed SereneDB protocol |

## Acceleration strategies

| Strategy | Examples | Main idea |
|---|---|---|
| [Build an index on GPU, search on CPU](gpu-vector-search.md) | Qdrant, OpenSearch/AWS, Oracle | Accelerate expensive preparation while preserving the existing search path |
| [Keep the index on GPU and search there](gpu-vector-search.md) | FAISS GPU, cuVS, Milvus GPU | Amortize index loading across many queries |
| [Split search across CPU, GPU, and SSD](gpu-vector-search.md) | BANG, FusionANNS | Handle data that does not fit entirely in VRAM |
| [Offload SQL operators to GPU](gpu-sql-offload.md) | Velox-cuDF, PG-Strom, Spark RAPIDS | Retain the planner and replace part of execution |
| [Execute most of an analytical plan on GPU](gpu-sql-offload.md) | Sirius, HeavyDB, NVIDIA GQE | Reduce intermediate transfers and use the GPU throughout a query |
| [Call a GPU service from the database](gpu-services.md) | Snowflake SPCS, ClickHouse embeddings | Accelerate one computation: a model, batch processing, or user-defined function |

For the first implementation experiment, start with [CPU-build profiling and GPU-assisted construction](gpu-serenedb.md#initial-experiment-gpu-assisted-ivf-construction), using [LanceDB](gpu-candidates/lancedb.md) and [PGPU](gpu-candidates/pgpu.md) as references. The agreed resident GPU-query backend follows this experiment. Use the [candidate selection](gpu-candidates/README.md) and [measurement protocol](gpu-candidates/benchmarks.md) to keep construction and query results separate.
