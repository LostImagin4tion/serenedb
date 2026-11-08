# GPU offload for SQL and analytical processing

[Research index](README.md) · [Vector-search survey](gpu-vector-search.md) · [GPU-backed services](gpu-services.md)

This survey covers GPU execution of SQL operators and analytical plans: scans, joins, aggregation, fusion, and data movement between execution stages.

Speedup figures come from authors' and vendors' publications; no local measurements were performed for this survey.

## Systems and integration models

The key question is where to move work across the device boundary. Offloading one small operation is often insufficient: conversion and transfer costs can exceed compute savings.

| Project | Actual approach | What to study |
|---|---|---|
| **Velox-cuDF** | Replaces Velox operators with libcudf implementations and inserts conversions at CPU/GPU boundaries. | Integrating an existing library into an existing executor. [Implementation README](https://github.com/facebookincubator/velox/blob/f13e473508756cfd20670df5d44a4289b2463c66/velox/experimental/cudf/README.md) |
| **Velox Wave** | Turns supported operator sequences into generated CUDA kernels, using JIT and GPU decoding. | Fusion and reduced intermediate materialization. The component is experimental. [Velox GPU components](https://facebookincubator.github.io/velox/develop/gpu-acceleration.html) |
| **Sirius** | Integrates with DuckDB, using GPU execution and CUDA-X libraries; the current architecture includes GPU/host/disk memory management. | The boundary between an existing planner and GPU executor, residency, and spilling. [Sirius source](https://github.com/sirius-db/sirius) |
| **gpudb** | DuckDB extension with Metal/CUDA and resident columns; transparent SQL execution needs a client wrapper. | Measurable CPU/GPU selection, memory budgets, and result-return costs. [Source and benchmarks](https://github.com/singhpratech/duckdbgpumetaldbram) |
| **HeavyDB** | Columnar database with CPU/GPU query JIT compilation and multilevel caching. | A complete specialized engine rather than a single GPU function. [HeavyDB source](https://github.com/heavyai/heavydb) |
| **PG-Strom** | Extends PostgreSQL planning with GPU scan/join/aggregation; supports GPUDirect Storage. | Offloading in an existing transactional database and shortening the NVMe → GPU path. [Documentation](https://heterodb.github.io/pg-strom/) |
| **BlazingSQL** | SQL layer over cuDF returning a GPU DataFrame. | A historical example of SQL/GPU-ecosystem compatibility. The available repository uses `branch-21.08`; modern stack readiness must not be assumed. [Source](https://github.com/BlazingDB/blazingsql) |
| **Spark RAPIDS** | Plugin replaces supported Spark operations with GPU implementations. | Partial execution, CPU/GPU boundaries, and suitable workloads. [AWS comparisons](https://awslabs.github.io/data-on-eks/docs/benchmarks/spark-rapids-gpu-benchmark) |
| **Polars GPU** | cuDF-based GPU backend for the Lazy API. | Logical-plan reuse and I/O effects. [Documentation](https://docs.pola.rs/user-guide/gpu-support/) |
| **NVIDIA GQE** | Open proof-of-concept GPU SQL engine accepting Substrait plans. | Co-designing formats, transfers, decompression, and execution. [Code](https://github.com/rapidsai/gqe), [architecture](https://developer.nvidia.com/blog/designing-gpu-accelerated-query-engines-with-nvidia-gqe) |
| **Microsoft Fabric** | GPU acceleration of suitable SQL operations through a coprocessor; limited preview. | Introducing GPUs into a commercial warehouse. The page provides neither open code nor a reproducible result. [Preview documentation](https://learn.microsoft.com/en-us/fabric/data-warehouse/query-acceleration) |

Velox also has **Breeze**, portable GPU primitives, and **UCX exchange**, transfers between GPUs without intermediate CPU-memory staging. These operate at different levels: Breeze supports kernel development, UCX reduces exchange costs, and cuDF/Wave execute operators. [Component overview](https://facebookincubator.github.io/velox/develop/gpu-acceleration.html).

## What SQL benchmarks show, including losses

Several results are particularly useful for understanding the mechanisms:

- **Velox-cuDF / Presto:** for 21 of 22 queries at SF1000, published aggregate times are **1246 s on AMD 7965WX** and **133.8 s on RTX PRO 6000 Blackwell**. This is about 9.3×, comparing specific hardware configurations; the CPU did not complete Q21 in the standard configuration. [Methodology and results](https://developer.nvidia.com/blog/accelerating-large-scale-data-analytics-with-gpu-native-velox-and-nvidia-cudf/).
- **Sirius:** the CIDR 2026 paper reports improved TCO on TPC-H/ClickBench. Read it alongside the implementation version and hardware costs: current `main` already differs from the studied version. [Sirius paper](https://www.vldb.org/cidrdb/papers/2026/p12-yogatama.pdf).
- **gpudb:** a published case has an approximately **32×** faster GPU kernel but only a **1.5×** faster complete operation because it returns 75M groups. Loading and sorting resident data also have separate costs. This illustrates the gap between a kernel benchmark and the user's result. [Project tables](https://github.com/singhpratech/duckdbgpumetaldbram).
- **Spark RAPIDS:** in AWS's TPC-DS 1 TB benchmark, q93 takes **118.21 s on RAPIDS versus 14.47 s on Gluten/Velox**. Other queries do win on GPU. Comparing against an optimized CPU executor fundamentally changes the picture. [Complete results](https://awslabs.github.io/data-on-eks/docs/benchmarks/spark-rapids-gpu-benchmark).
- **Polars:** the developers report up to 13× on selected PDS-H queries with H100 PCIe versus Xeon W9-3495X, while noting no substantial gains on I/O-bound queries. [Polars GPU benchmark](https://pola.rs/posts/gpu-engine-release/).

Working HeavyDB and PG-Strom implementations are established. This survey does not include a single speedup figure that could fairly be placed alongside these results without a separate experiment.

## Other hardware offload: Amazon Redshift AQUA

A useful hardware-offload architecture, but the originally published design uses **FPGAs and Nitro ASICs**, rather than GPUs. Discuss it separately from GPU cases as a comparison for moving scan/filter/aggregation closer to data. [AWS publication](https://aws.amazon.com/about-aws/whats-new/2021/06/aqua-for-amazon-redshift-launches-in-three-additional-aws-regions/).

## Public result sites

[ClickBench](https://benchmark.clickhouse.com/) provides analytical SQL results and CPU/GPU filtering. For benchmark sites and interpretation rules, see [public results and leaderboards](gpu-candidates/benchmarks.md#public-results-and-leaderboards).
