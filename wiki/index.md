# Wiki index

**Scope:** LLM KV cache management and serving systems, with a current emphasis on sparse attention, ANNS indexing, and memory tiers. The ingested set contains two surveys and sixteen systems/algorithm papers.

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

### Sparse attention and vector retrieval

- [Quest](sources/quest.md) — query-aware KV page selection from per-page key bounds; ICML 2024.
- [DeepSeek-V3.2](sources/deepseek-v3-2.md) — native sparse attention with a trained lightweight token indexer; arXiv v1, 2025.
- [RetrievalAttention](sources/retrievalattention.md) — OOD-aware ANNS over KV keys with CPU/GPU co-execution; NeurIPS 2025.
- [RoarGraph](sources/roargraph.md) — query-distribution-aware projected graph for OOD ANNS; PVLDB 2024.
- [RetroInfer](sources/retroinfer.md) — attention-aware cluster index and GPU/CPU KV buffer manager; PVLDB 2026.

### CXL and sparse-attention serving

- [CXL-ANNS](sources/cxl-anns.md) — CXL memory-pool placement, graph prefetch, and near-memory collaborative ANNS; USENIX ATC 2023.
- [Beluga](sources/beluga.md) — shared CXL memory pool for multi-host KV cache management; SIGMOD 2026.
- [SAC](sources/sac.md) — sparse-attention top-k KV reads from disaggregated CXL; arXiv preprint 2026.
- [ECHO](sources/echo.md) — host KV offload and lossless prefetch for native sparse-attention serving; OSDI 2026.

## Topics

- [KV cache management and serving systems](topics/kv-cache-management.md) — two complementary taxonomies and a cross-paper comparison of seven systems.
- [Sparse attention, CXL, and ANNS](topics/sparse-attention-cxl-anns.md) — the links and open design questions across sparse selection, approximate indexes, and memory tiers.

## Entities

_No standalone entity pages yet._

## Analysis

_No saved query analyses yet._

## Raw queue

All PDFs currently in `raw/` and `raw/papers/` have a source note. Newly collected system papers are cataloged in [raw/papers/README.md](../raw/papers/README.md).
