# Concurrent IVF: insert and search together

**Question:** What is the strongest current IVF reference when queries overlap with inserts, and what does that imply for a CPU+CXL design?

## Evidence from the two ingested papers

| Work | Scope | What it establishes | Boundary |
|---|---|---|---|
| [HAKES-Index](../sources/hakes.md), PVLDB 2025 | CPU-oriented embedding-search index and distributed database | Measures actual concurrent read-write execution with 32 clients, varies write ratio, and reports the comparison at recall 0.99. Inserts append encoded vectors to IVF partitions. | It is a compressed filter-and-refine design, not stock IVFFlat. It does not evaluate CXL. Deletion uses tombstones. |
| [SIVF](../sources/sivf.md), HPDC 2026 | GPU-resident streaming IVF | Specifies concurrent insertion and search safety using slot reservation and publish-after-initialize validity bits; also supports deletion. | CUDA/VRAM-specific; its results do not establish CPU+CXL performance. The archived PDF is the arXiv v3 manuscript. |

There is no single hardware-independent SOTA. For CPU-resident IVF-style concurrent read/write, HAKES-Index is the closest directly measured published baseline found in this research pass. For GPU-resident streaming, SIVF is the more directly relevant published design, but it targets VRAM rather than host or CXL memory.

## Distinguish concurrency from dynamic maintenance

Throughput under a mixed read/write trace is not by itself proof that readers and writers overlap safely. HAKES measures a concurrent workload and describes partition locking for its FAISS IVF baselines. SIVF additionally specifies the visibility protocol that prevents readers from consuming partially initialized entries.

Quake is relevant to a different axis: adaptive IVF partition maintenance under changing and skewed data. Its OSDI 2025 paper states that the current implementation serializes searches, updates, and maintenance; concurrent readers with background copy-on-write are discussed as a possible extension. This makes Quake a dynamic-maintenance comparison, not direct evidence of concurrent insert/search serving. [Quake, OSDI 2025](https://www.usenix.org/system/files/osdi25-mohoney.pdf)

## Implications for CPU+CXL research

For a memory-only CXL node, start with HAKES-style concurrent IVF and a partition-locked FAISS IVF baseline. Treat SIVF as a design reference for safe append publication, not as a performance baseline for CXL.

The research question suggested by these results is whether IVF lists can remain searchable during append and maintenance when the vector payloads reside on CXL and routing metadata may be kept in DRAM. The important measurements are query throughput and P99 latency, insertion latency, recall, per-list skew, synchronization/compaction pauses, local DRAM metadata footprint, and CXL traffic. This is a hypothesis for evaluation, not a novelty claim established by these papers.

The distinction matters for the wider sparse-attention proposal: CXL-Vector and CXL-ANNS establish CXL-side ANNS designs, while these two papers add concurrent and streaming IVF evidence. Combining them for KV selection still requires measuring the attention-query distribution, retrieval quality, and the cost of remote list access.
