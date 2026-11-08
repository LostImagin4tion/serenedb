# DuckDB FAISS extension: a close example of a SQL adapter

**Priority: primary as an integration example; GPU-path readiness needs independent verification.** This is a separate community extension, not a built-in DuckDB VSS capability. Checked on October 28, 2026.

## GPU role and integration boundary

The extension can create a named FAISS index, train it, add vectors, and search through SQL functions. Supported types can be moved to a GPU. The index object lives separately from the table's ordinary SQL index. [Project overview](https://github.com/duckdb-faiss-ext/duckdb-faiss-ext).

The inspected code creates a FAISS index through a factory and stores it in DuckDB's `ObjectCache`. Search retrieves the stored object, converts input values, and constructs the result. This illustrates the “SQL data → library index → SQL result” boundary, but does not by itself synchronize the index with table changes. [Main adapter, commit `c8fb12d`](https://github.com/duckdb-faiss-ext/duckdb-faiss-ext/blob/c8fb12d213fc232c98c61f8b230cfa13c52c6f26/src/faiss_extension.cpp).

The GPU helper calls `faiss::gpu::index_cpu_to_gpu()`. The inspected `innerCreateSearchParametersGPU()` creates basic IVF search parameters but does not forward all CPU parameters and the selector from its arguments. Therefore, exposing a SQL parameter does not establish support for it on the GPU path. This is a source-code observation, without a reproduced run. [GPU adapter](https://github.com/duckdb-faiss-ext/duckdb-faiss-ext/blob/c8fb12d213fc232c98c61f8b230cfa13c52c6f26/src/gpu/gpu.cpp).

## What results are actually published?

The README links to Jaap Aarts's 2024 thesis and newer VSS comparisons. The thesis explores filtered KNN strategies, including IVF2048, IVF65536, and HNSW128. The main testbed is an i9-9900K with 64 GB RAM; a separate VSS comparison uses a dual-socket Xeon E5-2670 with 256 GB. Both single-threaded and multithreaded CPU modes are covered. **These are CPU experiments; they do not establish GPU speedup.** [Methodology, thesis sections 3–4](https://www.cs.ru.nl/bachelors-theses/2024/Jaap_Aarts___1056714___Implementing_in-Database_Similarity_Search.pdf).

The project description reports benefits from batched execution, but some figures lack a displayed result and complete experimental configuration. No GPU speedup is therefore included in the comparison table here. [README and authors' qualifications](https://github.com/duckdb-faiss-ext/duckdb-faiss-ext).

**Evidence level:** source code and research on SQL integration are available; the checked materials did not establish a reproducible numerical GPU result. That does not establish an absence of benefit: it needs measurement.

## Relevance to the thesis

Study how train/add/search, IVF parameters, result lists, and filters are represented. SereneDB particularly needs the next step: library IDs must identify documents from the correct snapshot, and closing or replacing an index must release GPU resources.

Our conclusion: reuse the small adapter boundary and test scenario, but a `FAISS_SEARCH` function alone does not complete the work. The [thesis requirements](../../serenedb_gpu_diploma.md) call for SereneDB's actual server path, its index, correct ID mapping, and GPU-copy invalidation.

The README also describes retaining source data for retraining on `add` when the index has not been trained manually. A resident server backend must account for this memory duplication separately. [Training modes](https://github.com/duckdb-faiss-ext/duckdb-faiss-ext).

## Effort and proposed experiment

**Estimate:** a small SQL → FAISS proof of concept is low/medium effort; correct integration with the catalog and IResearch is high effort. DuckDB compatibility does not mean a third-party extension can be included unchanged in SereneDB's DuckDB fork.

Useful check: execute the same trained IVF-Flat directly in FAISS and through the SQL wrapper, separately for batch=1 and batched calls. Verify that `nprobe` actually changes, IDs and distances are correct, and execution uses the GPU. The time difference exposes conversion, dispatch, and result-construction costs. Confirm these properties before using the extension as a baseline.
