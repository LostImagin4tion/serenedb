# LanceDB / Lance: GPU construction of IVF indexes

**Priority: primary for the initial construction experiment.** Close to IVF and useful for its separation of GPU training from the storage format. Sources checked on October 28, 2026; its implementation priority follows the construction-first sequence.

## How and why GPUs are used

LanceDB added GPU-based IVF training through PyTorch, supporting CUDA and Apple MPS. In the original scenario, the GPU computes clusters, then returns centroids to the CPU/Rust side for index construction. This accelerates building; it does not establish GPU serving of ANN queries. [Original 2023 publication](https://www.lancedb.com/blog/gpu-accelerated-indexing-in-lancedb-27558fa7eee5).

The inspected modern Lance code contains `train_ivf_centroids_on_accelerator`, PQ training, partition assignment, and combined IVF/PQ train/assign helpers. The IVF helper selects CUDA/MPS, calls PyTorch KMeans, and returns centroids through `.cpu().numpy()`. Assignment reads data in batches and moves it to the device. [Code, commit `9953e71`](https://github.com/lance-format/lance/blob/9953e71827d9ff2b2e2dfdd91d716378f9dea6a6/python/python/lance/vector.py).

Consequently, the 2023 article's “IVF training only for now” limitation is not a complete description of current code. The newer functions also do not automatically establish a GPU query path; that needs separate verification.

## What the benchmark shows

The 2023 article measures **k-means training of IVF4096**, L2, 1M × 1536, OpenAI Ada2 embeddings:

| Testbed | CPU | GPU | Runtime ratio |
|---|---:|---:|---:|
| GCP g2-standard-32, NVIDIA L4 | 323 s | 12.5 s | ≈25.8× |
| M2 Max, 64GB, 30 GPU cores | 397 s | 21 s | ≈18.9× |

This is one stage; these ratios do not equal complete-build speedup and do not describe search latency. The article uses the general range “20–26×”; the table above retains the specific original timings. The active CPU-thread count and complete training configuration need clarification before reproduction. [Authors' conditions and results](https://www.lancedb.com/blog/gpu-accelerated-indexing-in-lancedb-27558fa7eee5).

## Architectural notes

The useful feature is a stable interface between stages: compute centroids on one device, then use an ordinary index representation. The server need not retain a GPU merely because the index was trained on one.

This approach has costs: PyTorch, batch loading, and Arrow/NumPy/Tensor conversion must enter the complete timer. Comparing CUDA and MPS helps assess the idea's portability, but Apple shared-memory MPS and a discrete CUDA GPU have different transfer costs. This is our architectural assessment; these overheads were not measured here.

**For SereneDB:** use this as an example without bringing Python/PyTorch into the server. Use the chosen stack's CUDA library capabilities while keeping each stage's inputs and outputs explicit. MPS is convenient for a learning prototype on a Mac, but the agreed thesis implementation remains Linux/NVIDIA CUDA.

## Relevance, effort, and experiment

**Estimate:** reproducing a separate training benchmark is low/medium effort; integrating GPU training into the existing builder is medium/high effort; resident SQL search is a separate task. This case does not replace the required GPU-search integration.

Useful thesis check: measure the complete build and the fraction originally spent on CPU training. If that fraction is small, even a large k-means speedup barely changes total build time. Compare recall of the resulting indexes as well as k-means inertia. Record the training sample, seed, iteration count, batch size, and normalization settings separately.

Read [PGPU](pgpu.md) alongside this: the two implementations independently show how to offload clustering to a GPU while retaining ordinary query serving. This is the recommended [initial SereneDB experiment](../gpu-serenedb.md#initial-experiment-gpu-assisted-ivf-construction), before the resident GPU-query integration.
