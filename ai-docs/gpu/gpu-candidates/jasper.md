# Jasper: updatable GPU Vamana with RaBitQ

**Priority: supplementary; a recent 2026 research example.** The inspected paper is **arXiv v5 dated July 31, 2026**, not the earlier v1 with different figures. Checked on October 28, 2026.

## What the GPU does

Jasper moves Vamana graph construction and search onto GPUs, adding batch insertions and RaBitQ. Its main idea is to reduce data movement and better hide irregular-access latency during graph traversal. [Paper v5](https://arxiv.org/html/2601.07048v5).

The published repository includes insertion and deletion, distinguishes soft deletion from compaction/consolidation, and provides C++/Python interfaces. These are interesting extensions to a static GPU snapshot, but do not remove the need for database transaction-visibility integration. [Jasper project](https://github.com/saltsystemslab/Jasper).

## Inspected code

`rabitq_index` stores GPU codes, a rotation matrix, and a transformed centroid. `build_rabitq_index` reads the graph under a shared lock, creates codes, and synchronizes completion. Codes correspond to the graph's physical slots. [RaBitQ index, commit `a0173f4`](https://github.com/saltsystemslab/Jasper/blob/a0173f41a0240ff52800b42320fa0132603f9e08/include/jasper/rabitq/index.cuh).

The documented search uses approximate distances first, then reranks against original vectors. Codes must be rebuilt after appends or slot renumbering; stable IDs and physical locations are separate concepts. [Pinned README](https://github.com/saltsystemslab/Jasper/blob/a0173f41a0240ff52800b42320fa0132603f9e08/README.md).

This is especially useful for SereneDB: even in a GPU library, stable IDs do not guarantee that every derived GPU structure is current.

## Metrics and published results

The main GPU is A100 80GB with CUDA 12.9; the CPU is Xeon Gold 5218 with 64 hardware threads. Datasets: BigANN100M, Deep100M, GIST1M, OpenAI-ArXiv 2.3M × 1536, and Text2Image10M. Metrics are build time, recall, and QPS.

| Dataset | Jasper build | CAGRA build |
|---|---:|---:|
| BigANN100M | 327.7 s | 1323.3 s |
| Deep100M | 296.2 s | 825.6 s |
| GIST1M | 12.2 s | 14.1 s |
| OpenAI | 89.0 s | 127.8 s |
| Text2Image | 24.3 s | 619.2 s |

The authors report up to **1.84×** query throughput over CAGRA. The variation in build gains shows why an average “7×” cannot be applied to every dataset. On Deep100M, CAGRA's build workspace affected algorithm selection; the comparison extends beyond search kernels. The quoted peak of 12.5M QPS on BigANN occurs at recall around **0.23** and is unsuitable as a high-quality search result. [Table 2 and section 6 of v5](https://arxiv.org/html/2601.07048v5).

## Important qualifications

The paper discusses CAGRA update limitations, but modern cuVS has `extend`. Compare specific versions and capabilities: insertion, deletion, rebuilding, and the cost of preserving quality are distinct properties. [Current CAGRA API](https://docs.nvidia.com/cuvs/user-guide/api-guides/indexing-guide/cagra).

Jasper's README examples use FP16, and the current RaBitQ path retains original vectors for reranking. “Up to 8× compression” therefore does not mean the **entire** resident index becomes 8× smaller. Account for the graph, originals, codes, and workspace. This qualification follows from the inspected API and code, without a local memory measurement.

## Relevance and effort for the thesis

**Estimate:** studying and reproducing a standalone benchmark is medium effort; integrating an updatable graph and quantization is very high effort, beyond the IVF MVP. Adding graphs or RaBitQ to the required implementation is not proposed.

Reuse the analysis method: distinguish bandwidth-bound from compute-bound stages and record peak workspace, quality, and vector type. For the first FP32 IVF implementation, this helps identify whether candidate reads, arithmetic, or insufficient parallel work limit performance.

If quantization is explored after the MVP, measure complete search including rotation and reranking at matched recall, separately from searching codes alone. Its direct contribution to the main version is methodology and a derived-structure invalidation contract, rather than adoption of a new algorithm.
