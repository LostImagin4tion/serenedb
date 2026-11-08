# pg_cuvs: GPU search through ordinary SQL

**Priority: primary as an architectural example; an experimental project, not a production backend validated by us.** Checked on October 28, 2026.

## What the GPU does

This PostgreSQL extension uses cuVS for resident search and a separate GPU-construction scenario followed by CPU HNSW serving. It preserves pgvector's operator interface. The GPU returns candidates; the SQL executor handles row retrieval and visibility. [Project overview](https://github.com/pg-cuvs/pg_cuvs).

This closely matches the thesis: accelerating part of an actual database query. However, the README's strong claims about complete semantics and production readiness cannot be established by the presence of code alone.

## Architecture and inspected code

PostgreSQL uses separate backend processes; the project provides a GPU daemon they access through IPC. It reuses the index and CUDA resources, avoiding a GPU context in every client process. [Project architecture](https://github.com/pg-cuvs/pg_cuvs).

The inspected wrapper contains actual CAGRA build/search/extend, IVF-PQ, and GPU-filter calls. At the bitmap boundary, bit semantics change: exclusion in the adapter becomes inclusion in cuVS. This is a concrete compatibility pitfall worth preventing with a test. [CUDA wrapper, commit `1c76a9d`](https://github.com/pg-cuvs/pg_cuvs/blob/1c76a9d4e150f4fb763b66488a21a3254c48ad5a/src/cuvs_wrapper.cu).

The SQL side fetches rows through `table_tuple_fetch_row_version()` with a transaction snapshot, skips invisible rows, and handles tombstones with snapshot awareness. This confirms work on semantics, but is not our complete correctness validation. [SQL access method](https://github.com/pg-cuvs/pg_cuvs/blob/1c76a9d4e150f4fb763b66488a21a3254c48ad5a/src/pg_cuvs.c).

**For SereneDB, this is an analogy rather than a ready-made design:** PostgreSQL's process model is a specific reason for the daemon. SereneDB should first assess an in-process adapter; IPC needs its own justification.

## Benchmark and actual timing boundaries

The published wiki_all_1M configuration uses 1M × 768, cosine, PostgreSQL 16.14, cuVS 26.04, and A100-SXM4-80GB. For Q=1, the authors report approximately 459.4 µs for GPU kernels, 1.44 µs for memcpy, and 882.1 µs of unprofiled daemon wall time. **The last figure is still not complete SQL latency.** Profiling significantly increased wall time; ratios mixing measurement modes require care. [Methodology and profiling](https://github.com/pg-cuvs/pg_cuvs/blob/1c76a9d4e150f4fb763b66488a21a3254c48ad5a/BENCHMARK.md).

The source CSV contains these latency-axis points for k=10 and 2000 queries:

| Path | Recall@10 | p50 | p95 |
|---|---:|---:|---:|
| pgvector HNSW, M=16, efConstruction=64, efSearch=200 | 0.9594 | 9.784 ms | 13.348 ms |
| pg_cuvs CAGRA, graph_degree=32, search parameter=200 | 0.9928 | 1.714 ms | 1.858 ms |

These are examples of published points, both above the 0.95 threshold but with different actual recall. They demonstrate possible gains, not a universal multiplier. [Raw results](https://github.com/pg-cuvs/pg_cuvs/blob/1c76a9d4e150f4fb763b66488a21a3254c48ad5a/bench/results/pg_cuvsbench_98.csv).

The authors explicitly warn that CPU and GPU concurrency rows have different recall, so peak QPS cannot be presented as iso-recall speedup. They also record an unsuccessful CAGRA scenario on 50M × 384: working data and build workspace did not fit in 2 × 40GB. [Comparison limitations](https://github.com/pg-cuvs/pg_cuvs/blob/1c76a9d4e150f4fb763b66488a21a3254c48ad5a/BENCHMARK.md).

## Relevance to the thesis

Reuse the latency breakdown: client → SQL → adapter → GPU → candidates → rows → client. Separate raw-cuVS and SQL baselines, execution-path diagnostics, ID checks, and explicit filter-support conditions are useful. A crossover point on synthetic data must not be transferred to real embeddings without measurements.

**Effort estimate:** a standalone cuVS call is low/medium effort; a similar server architecture is high effort. It includes IPC, process failures, index persistence, data visibility, and updates. An immutable snapshot substantially reduces thesis scope; automatic CPU fallback conflicts with explicit backend selection.

Proposed experiment: on one index and query set, measure warm library search, warm server search, and document-retrieval cost. Record queuing and errors separately from successful-query latency. This identifies where the accelerated kernel's benefits end.
