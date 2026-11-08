# VecFlow: GPU search with label filters

**Priority: secondary for the MVP, primary for filtered ANN research.** This is a GPU search backend as well as a construction accelerator. Checked on **October 31, 2026**. No local build or measurements were performed.

Paper: *VecFlow: A High-Performance Vector Data Management System for Filtered-Search on GPUs*, DOI [10.1145/3749189](https://doi.org/10.1145/3749189). The ACM PDF could not be opened directly; the inspected text is the [authors' arXiv v1 dated June 1, 2025](https://arxiv.org/html/2506.00812v1), linked from a [coauthor's page](https://moccch.github.io/publication/2026-sigmod-vecflow). Exact agreement with the publisher's PDF was not verified. Code is pinned to `b5d1cbe433e3b632d5e5758209fa2e5f6f73cd7d`, dated May 31, 2026.

## The key difference from ordinary IVF

VecFlow organizes posting lists **by label**, rather than proximity to centroids. Long lists use graph search; short lists use brute-force. Graph-side vectors can be shared between lists through local/global ID mapping; scanning uses an interleaved layout. [Authors' architecture description](https://supercomputing-system-ai-lab.github.io/projects/vecflow/filtered/).

The IVF name therefore does not imply compatibility with the existing IResearch IVF-Flat. In VecFlow, a query label directly selects a list; ordinary IVF finds the nearest coarse clusters. This difference determines both applicability and construction cost.

In the inspected builder, the graph branch is selected when `label_freq > specificity_threshold`; otherwise it uses BFS. Here BFS means **brute-force search**, not breadth-first graph traversal. The builder creates label-specific graphs and mappings; search classifies queries, runs CAGRA/scans, and gathers results. [Construction](https://github.com/Supercomputing-System-AI-Lab/VecFlow/blob/b5d1cbe433e3b632d5e5758209fa2e5f6f73cd7d/cpp/src/neighbors/vecflow/vecflow_build.cuh), [search](https://github.com/Supercomputing-System-AI-Lab/VecFlow/blob/b5d1cbe433e3b632d5e5758209fa2e5f6f73cd7d/cpp/src/neighbors/vecflow/vecflow_search.cuh).

The authors' high/low specificity terminology relates to the number of objects with a label. For our measurements, record `N_label` and the eligible-row fraction `s = N_label / N` to avoid ambiguity around “selectivity.”

## What the paper shows

The paper uses A100 40GB and an AMD EPYC 7763 CPU testbed listed as 128 cores/256GB RAM; GH200 is examined separately. Metrics are filtered recall@10, QPS, batch latency, and memory. The threshold is T=2000 and VecFlow graph degree R=16.

| Dataset | Data and filters | Result at recall 0.90 |
|---|---|---:|
| SIFT1M | 128D, 50 synthetic Zipf-distributed labels | 5M QPS versus StitchedVamana's 37K, about 135× |
| YFCC10M | 192D, real labels | 2.6M QPS; CPU IVF²: 48K |
| WIKI-1M | Wiki-ANN subset, 768D, two-label AND | 150K QPS |

Full Wiki-ANN contains 35M passages, but this experiment uses 1M because of VRAM. A separate persistent-kernel ablation reports up to 7.08× throughput and up to a 1.82× latency reduction over ordinary launches. These are different experiments with different baselines. [Preprint section 5](https://arxiv.org/html/2506.00812v1).

**Interpretation limits:** these are filtered-ANN results, not ordinary-IVF or SQL acceleration. CAGRA-Post/Inline are this work's baseline implementations, not an exhaustive assessment of modern cuVS. Reproducing the main QPS plot still requires pinning the query batch, actual CPU-thread usage, and included transfers; CPU-machine specifications alone do not establish these.

## What the available example measures

The README shows a GH200/SIFT1M example: `itopk=32`, about **6.04M QPS**, `avg=1.657 ms`, recall 0.9397. This is illustrative output from the current SDK, not the paper's 5M-QPS experiment. [Examples README](https://github.com/Supercomputing-System-AI-Lab/VecFlow/blob/b5d1cbe433e3b632d5e5758209fa2e5f6f73cd7d/vecflow/README.md).

The inspected C++ example loads query vectors and labels onto the device before timing; indexes, output buffers, ground truth, and warmup are also prepared in advance. It measures the `search + sync_stream` loop. Formulas:

```text
avg_ms = 1000 × total_time / num_runs
QPS = num_runs × N_query_vectors / total_time
```

Thus, `avg_ms` describes the **entire batch**; initial query upload and returning answers to the client are outside this timer. It includes orchestration inside `search`, so describing it as pure CUDA kernel time is also incorrect. [Timer and data preparation](https://github.com/Supercomputing-System-AI-Lab/VecFlow/blob/b5d1cbe433e3b632d5e5758209fa2e5f6f73cd7d/vecflow/examples/cpp/src/vecflow_example.cu).

The Python wrapper creates device buffers on every call, copies queries/labels, and returns NumPy results. Its wall time has a different boundary. [Wrapper](https://github.com/Supercomputing-System-AI-Lab/VecFlow/blob/b5d1cbe433e3b632d5e5758209fa2e5f6f73cd7d/vecflow/src/vecflow.cu).

## Differences between the paper, README, and API

- The inspected composite API supports single-label queries and a separate **two-label AND** path. The paper's AND/OR discussion does not establish arbitrary SQL-predicate support. The AND path selects the rarer label and checks the second during search. [Implementation](https://github.com/Supercomputing-System-AI-Lab/VecFlow/blob/b5d1cbe433e3b632d5e5758209fa2e5f6f73cd7d/cpp/src/neighbors/vecflow/vecflow_search_multi.cuh).
- The README describes a `≥ T` boundary; the inspected builder and query classifier use `> T`. This reproducibility setting must be checked against code, including `N_label = T`. [Classifier](https://github.com/Supercomputing-System-AI-Lab/VecFlow/blob/b5d1cbe433e3b632d5e5758209fa2e5f6f73cd7d/cpp/src/neighbors/vecflow/vecflow_common.cuh).
- The paper describes persistent kernels; ordinary composite `search()` does not enable that mode itself. The repository has a separate CAGRA persistent example. Its ordinary and persistent branches use different client-thread counts, so it does not isolate the kernel's effect alone. [Example](https://github.com/Supercomputing-System-AI-Lab/VecFlow/blob/b5d1cbe433e3b632d5e5758209fa2e5f6f73cd7d/examples/cpp/src/cagra_persistent_example.cu).
- This is a modified cuVS. The package ships its own `libcuvs.so`; the README warns of a conflict with stock cuVS in one conda environment. It cannot be treated as an ordinary independent library on top of any installed FAISS/cuVS. [Build and compatibility](https://github.com/Supercomputing-System-AI-Lab/VecFlow/blob/b5d1cbe433e3b632d5e5758209fa2e5f6f73cd7d/vecflow/README.md).

## Relevance to the thesis

The most applicable idea is an **exact GPU scan of a small eligible subset**. Instead of general ANN followed by discarded results, investigate passing eligible IDs, reading the corresponding FP32 vectors, and computing top-K. Measure subset formation and transfer costs: a small GPU scan does not guarantee fast complete SQL.

The second lesson is ID mapping: one document can belong to multiple posting lists, but results must expose a stable document ID. Label changes can invalidate the GPU index even when the vector is unchanged.

**Our effort estimate:** benchmarking existing VecFlow is medium effort; porting label-centric graphs is high effort; a server-side persistent queue with cancellation, fairness, and resource management is high effort. None is required for the MVP. Keep resident FAISS IVF-Flat without arbitrary filters first, then consider a small filtered-scan experiment as an extension.

## Benchmark ideas to reuse

The AND-query generator selects the second label by intersection size; the default is `min-and-size=50`, while the README example uses 500. If no suitable pair exists, the query is excluded from that run. Publish the threshold, seed, skipped-query count, and intersection-size distribution. [Generator](https://github.com/Supercomputing-System-AI-Lab/VecFlow/blob/b5d1cbe433e3b632d5e5758209fa2e5f6f73cd7d/vecflow/examples/python/generate_multi_query.py).

A reusable SereneDB protocol should independently sweep eligible-subset size, labels per document, and label/geometry correlation; use exact ground truth within the filter; separate rare, frequent, and AND queries; and report complete SQL latency alongside device-resident batch QPS. See [benchmarks.md](benchmarks.md).

## A separate system in the same repository

**VecFlow-Chamfer** belongs to another work, DOI [10.1145/3786706](https://doi.org/10.1145/3786706). It is ColBERT-style multi-vector retrieval with candidate generation and Chamfer reranking. Its memory assumptions, particularly GPU access to host RAM on a Superchip, do not transfer to an ordinary PCIe GPU. The documentation separates pipeline and raw-kernel benchmarks. Only the API scope was checked here; the second paper was not fully analyzed. [VecFlow-Chamfer README](https://github.com/Supercomputing-System-AI-Lab/VecFlow/blob/b5d1cbe433e3b632d5e5758209fa2e5f6f73cd7d/vecflow-chamfer/README.md).
