# Applying GPU research to SereneDB

[Research index](README.md) · [Vector-search survey](gpu-vector-search.md) · [Thesis requirements](../serenedb_gpu_diploma.md) · [Measurement protocols](gpu-candidates/benchmarks.md#initial-construction-experiment)

These are survey-based recommendations and proposed experiments. They do not change the agreed thesis scope or report a validated SereneDB GPU implementation.

## Where the gains come from

Several mechanisms recur across these implementations:

1. **Reusing data in VRAM.** Later queries amortize the initial transfer.
2. **Parallelism across candidates and queries.** Larger batches improve GPU utilization.
3. **High memory bandwidth.** Important for reading vectors and computing distances.
4. **Reducing transferred data.** Filtering, aggregation, and top-K happen before returning to the CPU.
5. **Fusion.** Several operations run without writing large intermediate results.
6. **Overlapping transfers and computation.** The next block loads while the current block is processed.
7. **Adapting algorithms to the device.** CAGRA illustrates this approach.

This synthesizes the surveyed sources. The inverse explains losses: too little work, large results, frequent transfers, insufficient VRAM, unsuitable algorithms, and weak executor integration.

## Applying this to SereneDB for a competitive result

In the current SereneDB code, IResearch organizes IVF with its own reader, writer, and centroid tree; FAISS provides individual components. Calling `index_cpu_to_gpu()` directly on an existing IResearch index is therefore impossible. It needs a supported FAISS object; data and IDs require adaptation. See [ivf_reader.cpp](../../iresearch/formats/ivf/ivf_reader.cpp) and [ivf_writer.cpp](../../iresearch/formats/ivf/ivf_writer.cpp).

### Initial experiment: GPU-assisted IVF construction

Start by profiling the existing CPU build, then test GPU offload of its costly training work. [LanceDB](gpu-candidates/lancedb.md) and [PGPU](gpu-candidates/pgpu.md) are the primary references for this experiment. The intended boundary is GPU centroid training, and possibly vector-to-cluster assignment, with results returned to the existing writer for ordinary CPU serving. This is a proposed integration, not an implemented capability.

The current [training entry point](../../iresearch/formats/ivf/clustering.cpp) selects Lloyd or SuperKMeans, and the [centroid builder](../../iresearch/formats/ivf/centroids.cpp) trains a hierarchy. Use that optimized CPU path as the baseline. Inspect the distribution of training-task sizes: offloading many small tasks may spend more time on transfers and launches than it saves.

The first experiment should establish:

- Time spent reading/sampling, training centroids, assigning vectors, quantizing/encoding and writing the index, plus GPU setup and transfers.
- Complete build wall time with the same source data, precision, metric, quantizer and recorded training settings.
- Recall and CPU-search performance of the resulting index; faster construction must be assessed together with quality.
- Peak RAM/VRAM and which stages actually executed on the GPU.

Whole-build acceleration is a hypothesis to test. As an illustrative calculation, if training is 20% of CPU build time, making that stage 10× faster leaves 82% of the original total time: about 1.22× overall speedup before additional overhead. These fractions are hypothetical, not SereneDB measurements.

Keep this experiment bounded to the profiled build stages. Its result informs the subsequent GPU-query integration even if it shows no speedup; it does not replace the agreed server-side GPU-search deliverable. The measurement details are in [the construction protocol](gpu-candidates/benchmarks.md#initial-construction-experiment).

### Subsequent stage: resident GPU queries

Recommendations for the GPU-query backend:

- **Keep the query MVP to resident IVF-Flat.** One GPU, one metric, and an immutable snapshot. This follows the construction experiment and matches the agreed GPU-search scope.
- **Use existing FAISS/cuVS GPU search.** Custom CUDA kernels should follow profiling of a specific bottleneck.
- **Keep the GPU copy across queries.** Associate it with a source-index version; changes must invalidate the copy.
- **Return primarily top-K IDs and distances from the GPU.** Returning all candidates moves substantial work and traffic back to the CPU.
- **Verify filters and document visibility.** Retrieving K neighbors and then applying a SQL filter may return fewer than K results and reduce quality. Filter support depends on the selected API and algorithm.
- **Design GPU-resource ownership.** Account for FAISS threading constraints and the full cost of temporary buffers.
- **Investigate batches and concurrency.** Batching can raise throughput, but waiting to fill a batch increases latency. Measure both.
- **Keep CPU/GPU selection explicit.** Hidden CPU fallback is unacceptable for thesis measurements because it obscures what is being measured.

These points are survey-based recommendations, not an already validated SereneDB GPU-backend architecture.

## How to demonstrate competitive performance

Evaluate construction first using complete build time and the resulting index's quality. The comparisons below then evaluate GPU query serving.

The sources do not support a universal promise that GPUs are always faster. A testable objective is to identify workloads where GPUs win at the same result quality.

This requires three measurement levels:

| Level | What it establishes |
|---|---|
| FAISS CPU / FAISS GPU / FAISS-cuVS | Library and hardware capabilities |
| Existing IResearch CPU / new GPU path | Adapter overhead and algorithm differences |
| Complete SQL through the server | Actual user benefit |

The minimum matrix should vary index size, dimensionality, K, probes, batch size, client count, and filter selectivity. Compare **at the same target recall**, with exact ground truth and a tuned multithreaded CPU baseline.

Measure separately:

- index preparation, training, and loading;
- cold and resident search;
- data conversion, transfers, GPU computation, and result processing;
- p50/p95/p99, QPS, RAM/VRAM, and execution cost.

### When does GPU preparation pay off?

This model concerns a resident GPU-query copy. Construction offload is evaluated by complete build wall time, including its GPU setup and transfers; its benefit does not depend on later queries reusing a resident GPU copy.

A resident GPU index has an upfront cost: preparing its data, allocating resources, and uploading it. Subsequent queries can recover that cost if each complete warm GPU query saves time relative to the CPU path. The question is how many queries must reuse the prepared copy before the accumulated savings cover preparation.

For sequential queries, a simple model is:

```text
TotalCPU(Q) = Tprepare_CPU + Q × TCPU
TotalGPU(Q) = Tprepare_GPU + Q × TGPU_complete_warm_query

Tpreparation = Tprepare_GPU − Tprepare_CPU
```

Here, `Tpreparation` is the **additional preparation time of the GPU path**, rather than every operation performed before GPU search. If the CPU index already exists, it may consist mainly of conversion, GPU-specific setup, and index upload. If both paths require the same training or preprocessing, that shared work cancels in the comparison. Include extra training only if the GPU path actually needs it.

`TCPU` and `TGPU_complete_warm_query` are measured mean wall times for complete warm queries under comparable conditions: the same workload, result quality, and measurement boundary. The GPU time includes per-query packing, query transfers, dispatch, synchronization, result transfers, and document retrieval where applicable. Index upload belongs to preparation only when the copy is reused. Kernel time alone cannot substitute for complete-query time. Means are used because this model adds elapsed times; p50 and p99 must be reported separately.

Setting the two totals equal gives the approximate break-even query count:

```text
Qbreak-even ≈ Tpreparation / (TCPU − TGPU_complete_warm_query)
```

The denominator is the time saved by one warm query. For a positive preparation overhead, it must be positive: if each GPU query takes at least as long as CPU, sequential repetition cannot recover that overhead. At break-even the total times are equal; additional queries produce a net time saving. This is a wall-time model, not a hardware-cost calculation.

For illustration, assume an already prepared CPU index, **20 s of additional GPU preparation**, **5 ms per CPU query**, and **3 ms per complete warm GPU query**. These are hypothetical values, not SereneDB measurements:

```text
Saving per query = 5 ms − 3 ms = 2 ms
Qbreak-even = 20 s / 0.002 s = 10,000 queries
```

| Queries served | CPU total | GPU total, including additional preparation |
|---|---:|---:|
| 5,000 | 25 s | 35 s |
| 10,000 | 50 s | 50 s |
| 20,000 | 100 s | 80 s |

The relevant `Q` is the number of queries served **during the GPU copy's residency and validity**, not the database's lifetime query count. Invalidation, unloading, or eviction can require preparation again. Evaluate the cost actually incurred in each reuse window. A faster warm query can still be a poor choice if the copy is discarded before enough queries reuse it.

### Why asynchronous execution needs a different comparison

The sequential model assumes one complete query finishes before the next starts. A pipeline can overlap preparation of one query, GPU work on another, and result processing on a third. Consequently, summing individual stage times does not predict the elapsed time of the whole workload. In an ideal pipeline, steady throughput is bounded by the slowest stage; resource contention can lower it further.

A GPU path can have higher single-query latency and still finish a large concurrent workload sooner through greater aggregate throughput. That does not contradict the sequential formula: the execution model has changed. Both CPU and GPU paths must be measured at their stated concurrency and resource allocations.

For a throughput comparison, a rough corresponding model is:

```text
TotalCPU(Q) ≈ CCPU + Q / RCPU
TotalGPU(Q) ≈ CGPU + Q / RGPU

Qbreak-even ≈ (CGPU − CCPU) / (1 / RCPU − 1 / RGPU)
```

`RCPU` and `RGPU` are measured steady-state completion rates; `CCPU` and `CGPU` represent preparation and startup/drain overhead outside the steady-state term. This approximation needs validation against measured workload wall time, especially for short runs. Use consistent units: requests and requests/s, or query vectors and vectors/s. Batch throughput does not establish single-request latency; include batch-filling and queue waits when measuring p50/p95/p99.

For SereneDB, record cold preparation, warm query latency, steady throughput, and total wall time for a fixed query set separately. Stage timers explain the bottleneck; synchronized completion of the actual workload establishes whether preparation paid off.

Reading priority:

1. Construction experiment: [LanceDB](gpu-candidates/lancedb.md), [PGPU](gpu-candidates/pgpu.md), and the FAISS training/assignment APIs.
2. GPU-query integration: [FAISS GPU](gpu-candidates/faiss-gpu.md), [cuVS](gpu-candidates/cuvs.md), [DuckDB FAISS extension](gpu-candidates/duckdb-faiss.md), [pg_cuvs](gpu-candidates/pg-cuvs.md), and [Milvus](gpu-candidates/milvus.md).
3. Build/serve separation and background-load effects: [Elasticsearch](gpu-candidates/elasticsearch.md), [Qdrant](gpu-candidates/qdrant.md), OpenSearch, and Oracle in the [vector survey](gpu-vector-search.md).
4. Extensions: [RUMMY](gpu-candidates/rummy.md) for partial residency, [VecFlow](gpu-candidates/vecflow.md) for filtered search, and [Sirius/Velox](gpu-sql-offload.md) for SQL offload. GGNN and Jasper remain optional graph comparisons.

[Candidate index](gpu-candidates/README.md).
