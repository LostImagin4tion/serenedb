# PGPU and VectorChord: GPUs for IVF training

**Priority: primary for the initial construction experiment.** A close example of accelerating IVF construction while retaining CPU serving. Sources checked on October 28, 2026; its implementation priority follows the construction-first sequence.

## What the GPU does

PGPU is an EnterpriseDB PostgreSQL extension. It reads table vectors, computes centroids on a GPU, writes them to PostgreSQL, and starts VectorChord construction with external centroids. In this verified scenario, VectorChord serves queries; the GPU does not become the search backend. [PGPU overview](https://github.com/EnterpriseDB/pgpu).

The useful property is a small hardware-acceleration boundary: the GPU produces ordinary centroids. The storage format and server query path retain their existing owner. This does not satisfy the thesis's resident-search goal, but is a good example of an independent build optimization.

## Source-code evidence

The SQL function `create_vector_index_on_gpu` accepts sampling, batch, k-means iteration, metric, and normalization settings. [SQL entry point, commit `b39cc5e`](https://github.com/EnterpriseDB/pgpu/blob/b39cc5e3fa49efb673c524d722ea7c812228bc7e/src/vectorchord_indexing.rs).

The Rust implementation uses cuVS `kmeans::fit` and `predict`, moves data into a device ManagedTensor, and returns centroids and labels to host memory. Separate operations train root and leaf clusters and assign vectors to roots. This is actual GPU-accelerated clustering. [GPU clustering](https://github.com/EnterpriseDB/pgpu/blob/b39cc5e3fa49efb673c524d722ea7c812228bc7e/vectorchord-indexing/src/clustering_gpu_impl.rs).

**Our interpretation:** label assignment, result transfers, and SQL writes may become a significant fraction of runtime after accelerating k-means. A timer for the complete build operation is therefore necessary.

## Metrics and published gains

The inspected PGPU README has no complete CPU/GPU table with hardware, dataset, quality, and wall-clock build time. It does not support a defensible speedup multiplier. This is a gap in experimental evidence, not evidence of ineffectiveness.

EnterpriseDB also has the [Vector Search Benchmark Toolkit — vsbt](https://github.com/EnterpriseDB/vsbt). It includes PGPU/VectorChord scenarios, real-world and classic datasets, centroid reuse, and multiple SQL clients. It could support future build-time, QPS, and recall checks, but this research did not reproduce a numerical PGPU table.

An important counterexample from a related project: VectorChord 1.0 describes substantial **CPU** build acceleration through algorithm changes, including clustering dimensionality and hierarchy. A GPU must therefore not be compared only with a deliberately simple CPU k-means implementation, with that gain then attributed to all indexing. [VectorChord 1.0 discussion](https://blog.vectorchord.ai/vectorchord-10-developer-first-vector-search-on-postgres-100x-faster-indexing-than-pgvector).

## Lessons for the thesis

Separate centroid training, list assignment, and search serving. These operations can be measured and accelerated independently. Reusing CPU-index centroids would also allow GPU-search experiments without changing coarse-partitioning quality.

External-centroid support in another engine does not establish that SereneDB already exposes such an input. The [local IVF builder](../../../iresearch/formats/ivf/ivf_writer.cpp) and its contract need inspection; this research made no code changes.

## Effort and proposed experiment

**Estimate:** reusing the “GPU k-means → CPU serialization” idea is medium effort; a complete GPU query backend is a separate high-effort task. Adding Rust/PGPU as a SereneDB dependency is not proposed; the value is the architectural example and methodology.

Experiment: fix the seed, cluster count, iterations, metric, and normalization on one training sample. Measure sampling/reading, upload, fit, assignment, download, and the complete build separately. Check search recall afterward: shorter training is not equivalent if the resulting index is worse. The SereneDB baseline should use the existing CPU builder rather than an external teaching implementation. This is the recommended [initial experiment](../gpu-serenedb.md#initial-experiment-gpu-assisted-ivf-construction); the resident GPU-query stage follows it.
