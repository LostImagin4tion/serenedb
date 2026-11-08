# GGNN: an independent GPU graph-search implementation

**Priority: a supplementary baseline and source of methodological cautions.** This is a graph index, not a replacement for the required IVF. Checked on October 28, 2026.

## How the GPU is used

GGNN builds and traverses a nearest-neighbor graph in CUDA. It provides C++/CUDA and Python interfaces, accepts CPU/GPU tensors, and supports sharding across GPUs. In a single-GPU scenario, device data can be exchanged without an intermediate CPU copy. The project distinguishes the current library from `release_0.5`, which corresponds to the published work. [GGNN repository](https://github.com/cgtuebingen/ggnn).

It is a useful independent reference: fast GPU search extends beyond FAISS. A GGNN comparison, however, changes both algorithm and index structure, so it cannot be presented as pure device acceleration of the existing IVF.

## Implementation evidence

In the inspected CUDA query layer, a block selects a query through `blockIdx.x`, creates a working cache, loads starting points, and executes a bounded traversal loop. Results account for the shard ID. `tau_query` and `max_iterations` participate in search. [Query layer, commit `2d03b36`](https://github.com/cgtuebingen/ggnn/blob/2d03b36362bc824e37099092eabb95172ec18b64/src/ggnn/query/query_layer.cu).

The `ggnn_benchmark` utility accepts base/query/ground-truth data, saves the graph, and sweeps parameters. The documentation describes automatic swapping of shards between GPU, RAM, and disk when all shards do not fit. “Billion-scale” therefore does not mean an entire billion vectors stays on one GPU. [Benchmarking guide](https://ggnn.readthedocs.io/en/latest/benchmarking.html).

## Benchmarks and comparison limitations

A particularly useful independent HardBD & Active 2025 study compares SONG, CUHNSW, GANNS, GGNN 0.5, and CAGRA. Its testbed uses A100 PCIe 40GB, two Xeon E5-2640 v4 CPUs, and 128GB RAM; CPU HNSW is also tested with 40 threads. Main plots show top-10, Pareto recall/QPS, and batch=10,000, or 1000 for GIST.

On SIFT at recall >0.95, GGNN throughput at batch=10 is **121×** lower than at batch=10,000. This is GPU-utilization behavior within one algorithm, not CPU/GPU speedup. [Paper and methodology](https://www.shimin-chen.com/papers/gpu-graph-anns-hardbdactive25.pdf).

The [comparison's source code](https://github.com/schencoding/gpu-graph-anns) is available for further checks. It uses a common cuVS-based framework, which is more useful than comparing unrelated marketing tables.

**Not established:** the current GGNN library need not reproduce version 0.5's figures. These batch-QPS results do not determine single-query SQL latency. The checked materials do not establish a specific end-to-end speedup over SereneDB.

## Lessons for the thesis

Useful elements include explicit residency, query-budget control, and support for standard datasets. Before interpreting “time per query,” determine whether it is the wall time of an individual query or batch time divided by batch size.

Our proposal: use GGNN as an optional standalone baseline on the same FP32 data and metric if time remains after the main IVF experiment. Compare at recall 0.90/0.95/0.99 and batch=1/several/thousands. This helps distinguish limitations of IVF, the selected implementation, and GPUs in general.

## Effort

**Estimate:** a standalone library comparison is medium effort; server integration is high effort and outside the MVP. It requires separate graph construction, ID mapping, memory, and lifecycle management. Adding a second ANN backend to SereneDB for one comparison plot is unnecessary.

The first result should be a separate benchmark on shared data, without changing SereneDB's build/configuration. A complete graph-based SQL backend is justified only as a separate extended task.
