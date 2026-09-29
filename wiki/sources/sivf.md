# SIVF: GPU-Resident IVF Index for Streaming Vector Analytics

## Bibliographic details

- Author: Dongfang Zhao.
- Venue: HPDC 2026, pp. 31–44. DOI: [10.1145/3806645.3807575](https://doi.org/10.1145/3806645.3807575).
- Archived PDF: [`raw/papers/sivf-arxiv26.pdf`](../../raw/papers/sivf-arxiv26.pdf), arXiv v3 dated 26 March 2026. The proceedings title is “Streaming Vector Analytics”; the archived arXiv version uses “Streaming Vector Search.”
- Canonical paper and version history: [HPDC 2026 program](https://hpdc.sci.utah.edu/2026/program.html); [arXiv:2601.11808](https://arxiv.org/abs/2601.11808); [arXiv v3 PDF](https://arxiv.org/pdf/2601.11808v3).
- Implementation: [ElasticIVF](https://github.com/hpdic/ElasticIVF), integrated as a GPU Faiss index according to the paper.

## Summary

SIVF adapts IVF to streaming updates on a GPU. Instead of fixed contiguous inverted-list buffers, it stores each list as linked fixed-capacity slabs. Atomic slot reservation supports concurrent insertion; an address-translation table maps vector IDs to slots for deletion; a validity bitmap makes a slot visible to search only after its payload has been initialized.

## Key claims from the paper

- **Concurrent insertion (§3.2):** insertion threads reserve slots with atomic operations and publish completed vectors through a validity bit after a device memory fence. The paper states a linearizability result for parallel ingestion.
- **Search during updates (§3.3, §3.5):** search checks validity bits before reading entries. The paper proves its search path safe under concurrent ingestion and deletion, avoiding reads of partially initialized vectors.
- **Deletion (§3.4):** an ID-to-location table and atomic validity-bit clearing make logical deletion constant-time; empty slabs may be reclaimed through the slab pool.
- **Reported results (§5):** the evaluation compares with GPU IVF and other indexes on several datasets, including billion-scale cases, and reports 36–120× higher ingestion throughput than existing GPU indexes, up to 1765× lower deletion latency, and 4.07M inserts/s plus 108.5M deletes/s across 12 GPUs. These are paper-reported results on its evaluation setup.
- **Search quality (§5.3):** the authors report recall parity with the contiguous IVF baseline on their tested datasets. The slab layout is a data-structure change; it does not itself guarantee quality under distribution shift.

## Limitations and evidence boundary

SIVF is a GPU-resident design: its slab allocator, atomics, publication protocol, and measured update path are CUDA-specific. The paper does not evaluate CPU-side CXL memory or compare CXL access policies. Its updates assign vectors to the existing IVF lists; adaptive centroid retraining and CXL-aware rebalancing are not the contribution studied here. Linked slabs also trade contiguous scans for pointer traversal, so query throughput and memory behavior should be measured for each dimensionality and workload.

## Relevance to this wiki

SIVF is a peer-reviewed HPDC 2026 reference for concurrent publication and streaming maintenance on a GPU-resident path, including the available A100. It is not a direct baseline for the CPU+CXL memory-only path. See [Dynamic IVF and concurrent read-write](../analysis/concurrent-ivf-read-write.md) and [Sparse attention, CXL, and ANNS](../topics/sparse-attention-cxl-anns.md).
