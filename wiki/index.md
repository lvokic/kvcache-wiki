# Wiki index

**Scope:** LLM KV cache management and serving systems, with a current emphasis on sparse attention, ANNS indexing, and memory tiers. The ingested set contains two surveys and 40 systems/algorithm papers, including the owner's anonymized CXL-Vector submission manuscript.

## Overview

- [LLM KV Cache Wiki](overview.md) — current scope, working synthesis, and suggested reading paths.

## Sources

### Surveys

- [A Survey on Large Language Model Acceleration based on KV Cache Management](sources/survey-kv-cache-management.md) — broad token/model/system taxonomy; TMLR 2025.
- [From Tensor Buffer to Distributed Memory Hierarchy](sources/distributed-kv-hierarchy-survey.md) — locality, lifetime, ownership, and substrate taxonomy; arXiv v1, 2026.

### Systems papers

- [PagedAttention / vLLM](sources/pagedattention.md) — paged local GPU KV allocation; SOSP 2023.
- [DistServe](sources/distserve.md) — prefill/decode disaggregation and SLO-aware placement; OSDI 2024.
- [InfiniGen](sources/infinigen.md) — prediction-based KV prefetching for host offload; OSDI 2024.
- [CacheGen](sources/cachegen.md) — compressing and streaming KV over the network; SIGCOMM 2024.
- [vAttention](sources/vattention.md) — CUDA virtual memory for dynamic KV allocation; ASPLOS 2025.
- [Mooncake](sources/mooncake.md) — global KV cache, disaggregation, and tiered storage; FAST 2025.
- [FlashInfer](sources/flashinfer.md) — block-sparse KV layouts and a customizable attention engine; MLSys 2025.
- [LayerKV](sources/layerkv.md) — layer-wise KV allocation/offload with an SLO-aware scheduler; arXiv v3, 2024.
- [IceCache](sources/icecache.md) — semantic KV clustering, dynamic page mapping, and GQA-aware backload; ICLR 2026.
- [SPIN](sources/spin.md) — unified sparse-attention substrate and hierarchical KV management; arXiv 2026.
- [Fluxion](sources/fluxion.md) — output-aware sparse budgets and CPU/GPU hybrid attention scheduling; arXiv 2026.
- [HiSparse](sources/hisparse.md) — exact sparse-selection resolution over host KV with bounded GPU cache; arXiv 2026.
- [ScoutAttention](sources/scoutattention.md) — layer-ahead CPU precomputation for GPU/CPU KV offload; DAC 2026.
- [Strata](sources/strata.md) — hierarchical context caching and GPU-assisted I/O; OSDI 2026.
- [DirectKV](sources/directkv.md) — zero-copy KV offload on NVLink-C2C CPU/GPU systems; OSDI 2026.
- [SWARM](sources/swarm.md) — co-activation-aware placement across multiple SSDs; arXiv 2026.
- [Exploring CXL-based KV Cache Storage](sources/exploring-cxl-kv-storage.md) — CXL KV/prefix storage and serving economics; NeurIPS 2024 ML Systems workshop.

### Sparse attention and vector retrieval

- [MInference 1.0](sources/minference.md) — per-head dynamic sparse patterns and GPU kernels for long-context prefill; NeurIPS 2024.
- [Quest](sources/quest.md) — query-aware KV page selection from per-page key bounds; ICML 2024.
- [DeepSeek-V3.2](sources/deepseek-v3-2.md) — native sparse attention with a trained lightweight token indexer; arXiv v1, 2025.
- [RetrievalAttention](sources/retrievalattention.md) — OOD-aware ANNS over KV keys with CPU/GPU co-execution; NeurIPS 2025.
- [RoarGraph](sources/roargraph.md) — query-distribution-aware projected graph for OOD ANNS; PVLDB 2024.
- [RetroInfer](sources/retroinfer.md) — attention-aware cluster index and GPU/CPU KV buffer manager; PVLDB 2026.
- [CompactAttention](sources/compactattention.md) — GQA-aware block union and in-place KV tables for chunked prefill; arXiv 2026.
- [MiniMax Sparse Attention](sources/minimax-sparse-attention.md) — trained per-GQA-group block selector and KV-outer kernel; arXiv 2026.
- [Louver](sources/louver.md) — halfspace range-search index with a threshold-relative zero-false-negative guarantee; arXiv 2026.
- [Verified vAttention](sources/vattention-verified.md) — top-k plus sampling with user-specified approximation guarantees; ICLR 2026. Distinct from the CUDA VMM paper above.
- [Self-Indexing KVCache](sources/self-indexing-kvcache.md) — compressed sign-based keys double as the sparse retrieval structure; AAAI 2026.
- [SALS](sources/sals.md) — latent-space KV compression and sparse selection with RoPE-aware design; arXiv 2025.
- [FreshDiskANN](sources/freshdiskann.md) — concurrent graph-ANN insert/delete/search and streaming index maintenance; arXiv 2021.
- [HAKES](sources/hakes.md) — compressed IVF-style filter-and-refine index with measured concurrent read-write workloads; PVLDB 2025.
- [SIVF](sources/sivf.md) — GPU-resident IVF with concurrent streaming insertion, search, and deletion; HPDC 2026 (archived arXiv v3).

### CXL and sparse-attention serving

- [CXL-ANNS](sources/cxl-anns.md) — CXL memory-pool placement, graph prefetch, and near-memory collaborative ANNS; USENIX ATC 2023.
- [Beluga](sources/beluga.md) — shared CXL memory pool for multi-host KV cache management; SIGMOD 2026.
- [SAC](sources/sac.md) — sparse-attention top-k KV reads from disaggregated CXL; arXiv preprint 2026.
- [ECHO](sources/echo.md) — host KV offload and lossless prefetch for native sparse-attention serving; OSDI 2026.
- [CXL-Vector](sources/cxl-vector.md) — DRAM-resident graph/code navigation with original-vector reranking from memory-only CXL; owner-provided anonymized SIGMOD ’27 submission P1297.
- [COSMOS](sources/cosmos-cxl-anns.md) — general-purpose cores inside CXL devices for full in-memory ANNS; IEEE Computer Architecture Letters 2025.
- [PNM-KV](sources/pnm-kv.md) — CXL-attached processing-near-memory for KV selection and attention; PACT 2025.
- [TRACE](sources/trace-cxl.md) — bit-plane layout, lossless compression, and precision-proportional CXL fetch; arXiv v3, 2026.

## Topics

- [KV cache management and serving systems](topics/kv-cache-management.md) — two complementary taxonomies and a cross-paper comparison of seven systems.
- [Sparse attention, CXL, and ANNS](topics/sparse-attention-cxl-anns.md) — the links and open design questions across sparse selection, approximate indexes, and memory tiers.

## Entities

_No standalone entity pages yet._

## Analysis

- [Dynamic IVF and concurrent read-write](analysis/concurrent-ivf-read-write.md) — compares HAKES and SIVF, separates update maintenance from true concurrent query/insert serving, and maps the evidence to CPU+CXL.

## Raw queue

All 42 PDFs currently in `raw/papers/` have a source note. The complete paper catalog is [raw/papers/README.md](../raw/papers/README.md).
