# NVIDIA cuVS: IVF and CAGRA on GPUs

**Priority: primary for IVF and benchmarking; graph-based CAGRA is a comparison algorithm.** Checked on October 28, 2026.

## What the GPU does

cuVS provides GPU construction and search algorithms: IVF-Flat, IVF-PQ, CAGRA, and brute-force. SereneDB can explore the library through FAISS while retaining the selected integration boundary. Integrating cuVS directly would be a separate architectural decision. [cuVS repository](https://github.com/NVIDIA/cuvs).

IVF-Flat first selects the nearest coarse lists, then compares the query with vectors in those lists. In resident search, the main data stays in VRAM. `n_lists` affects the structure; `n_probes` affects scan volume and recall. Memory estimates must include IDs, list capacity, and temporary buffers in addition to `N × d × 4`. [IVF-Flat guide](https://docs.nvidia.com/cuvs/user-guide/api-guides/indexing-guide/ivf-flat).

CAGRA uses a GPU graph. Its structure and execution target GPU parallelism; different ways to distribute work across blocks are available for different batch sizes. It supports conversion to CPU HNSW and has a graph-extension API. Consequently, a paper's blanket claim that “CAGRA cannot be updated” should not be applied to every cuVS version. [CAGRA guide](https://docs.nvidia.com/cuvs/user-guide/api-guides/indexing-guide/cagra).

## Publications and benchmarks

In the original CAGRA paper, ICDE 2024, the authors report 2.2–27× faster construction than HNSW and 33–77× faster large-batch search at 90–95% recall. These results concern a specific version, hardware, and baseline set; they do not establish the same SQL speedup. [CAGRA paper](https://arxiv.org/abs/2308.15136).

It is more useful to retain both single-query performance and throughput. NVIDIA's 2025 FAISS/cuVS publication has separate plots for one query and batches of 10,000. For large batches, CAGRA beats CPU HNSW by up to 18× on the image dataset and over 8× on text embeddings, comparing Pareto points around recall 0.95. [NVIDIA benchmark](https://developer.nvidia.com/blog/enhancing-gpu-accelerated-vector-search-in-faiss-with-nvidia-cuvs).

These results show the potential of GPU graph search, but the thesis's IVF requires separate measurements. The classic IVF versus cuVS-IVF comparison through FAISS is covered in [faiss-gpu.md](faiss-gpu.md).

## Verified source code and API

cuVS has an extensible benchmark workflow: a dataset configuration, build/search parameter sets, and a backend that returns results. It can integrate another engine rather than only run built-in kernels. [Backend contracts, commit `281331f`](https://github.com/NVIDIA/cuvs/blob/281331f736886ced474459f552c76822cde03d09/fern/pages/cuvs_bench/pluggable_backend.md).

Distinguish this mechanism from the actual timer guarantees of a particular backend. A field named `end_to_end` does not by itself include index loading, networking, and SQL: the measurement boundary determines that. [cuVS Bench metrics](https://docs.nvidia.com/cuvs/user-guide/benchmarking-guide/cu-vs-bench-tool/usage).

## Filters and quality

An IVF filter restricts eligible neighbors but does not guarantee that the desired neighbor lies in the scanned lists. Restrictive filters may require more probes or another search strategy. Filtered recall must be compared with exact top-K **within the eligible subset**. [IVF filtering documentation](https://docs.nvidia.com/cuvs/user-guide/api-guides/indexing-guide/ivf-flat).

**Thesis recommendation:** initially reject unsupported filters explicitly. If a bitmap is added later, measure its construction and transfer costs together with search; it is not a free kernel parameter.

## What to reuse and implementation effort

**Estimate:** using cuVS through the existing FAISS API is medium effort once FAISS GPU works; a separate backend with new formats and resources is high effort. CAGRA needs a different index and is outside the required IVF scope.

The three most useful elements are a ready-made IVF implementation, explicit peak/steady VRAM accounting, and cuVS Bench. Experiment: vary `nprobe`, batch size, and concurrency on the same data, plot recall/latency/QPS curves, and compare with an optimized multithreaded CPU baseline. Do not select a winner from a single parameter configuration.
