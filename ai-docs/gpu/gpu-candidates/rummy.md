# RUMMY: IVF search on data larger than VRAM

**Priority: secondary; the research work closest to IVF with partial residency.** NSDI 2024, with available source code. Checked on October 28, 2026.

## Problem and data movement

RUMMY considers an index that fits in server RAM but not in a single GPU's VRAM. Instead of keeping every IVF list on the device, it fetches the required parts. A query batch can be regrouped around shared clusters to transfer them fewer times and overlap transfers with computation. [Authors' system overview](https://www.usenix.org/conference/nsdi24/presentation/zhang-zili-pipelining).

Data path:

```text
IVF lists in RAM + query batch
    → determine the clusters needed by queries
    → transfer order and processing groups
    → GPU buffers: loading / scanning / top-K
    → merge results
```

CUDA unified memory alone does not solve this problem: page migration does not know which IVF lists future queries will need. This explains the query-aware scheduler. [RUMMY motivation](https://www.usenix.org/conference/nsdi24/presentation/zhang-zili-pipelining).

## Source-code evidence

This is a modified FAISS, not a small wrapper around current cuVS. The inspected `PipeScheduler` calls `sample_list`, orders clusters, forms groups, and processes them with separate buffers and synchronization. It accounts for reordering/grouping and transfers separately. [Scheduler, commit `accf7f8`](https://github.com/pkusys/Rummy/blob/accf7f81996d67460654a7493183af952dc3ff28/faiss/pipe/PipeScheduler.cu).

Also inspect `IndexIVFPipe`, `PipeCluster`, `PipeProfiler`, and `RestrictedGpuResources` in the [pinned tree](https://github.com/pkusys/Rummy/tree/accf7f81996d67460654a7493183af952dc3ff28/faiss/pipe). These are the places to study list selection, work cost, residency, and memory limits. Their presence alone does not establish compatibility with a modern CUDA toolchain.

## Benchmarks and results

The main experiments use SIFT1B, DEEP1B, and TEXT1B, with 128, 96, and 200 dimensions. The GPU testbed is AWS p4d.24xlarge with 1152GB RAM, but RUMMY uses **one A100 40GB** out of eight; the CPU baseline is x1.16xlarge with 64 vCPUs and 976GB RAM. Batch sizes are **8 and 2048**, not single queries.

The authors report **2.4–23.1×** speedup over IVF-CPU. Against IVF with CUDA unified memory, the gains are **121–135×** for the large batch and **2.1–3.7×** for the small batch. The main metric is average end-to-end time per query; the comparison includes data larger than VRAM. Accuracy/time curves and ablations of individual mechanisms are provided. [Paper, section 6](https://www.usenix.org/system/files/nsdi24-zhang-zili-pipelining.pdf).

**Interpreting the gains:** the large multiplier over unified memory demonstrates removal of a poor migration regime, not the same advantage over resident GPU IVF. Per-query time at batch=2048 is not a single server query's p99. The paper's economic conclusions depend on historical prices and how a large GPU instance's cost is allocated.

## Relevance to the thesis

The key idea is a resource shared across queries: different queries may read the same IVF list. Transferring a list once and processing all interested queries changes PCIe traffic volume. This is not required for the resident MVP, but helps explain the boundary where an index no longer fits in VRAM.

Recommended future experiment: limit available VRAM and measure H2D transfer count and volume, list reuse, GPU utilization, and batch waiting time. Compare simple streaming with the scheduler while preserving data, lists, and quality. This ablation explains the mechanism behind a speedup as well as its magnitude.

## Implementation effort

**Estimate: high.** It requires memory-block management, pinned buffers, cost prediction, transfer/search coordination, and top-K merging. Porting an older research fork may be harder than reimplementing a limited mechanism on the selected backend.

The thesis MVP requires the index to fit in VRAM. RUMMY is therefore survey material and a possible extension after the main server path works, rather than a reason to add an out-of-core scheduler immediately.
