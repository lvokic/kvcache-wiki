# Wiki log

Append new entries below. Use the format `## [YYYY-MM-DD] kind | description`.

## [2026-09-29] setup | Wiki scaffold

- Added the operating guide, source handling notes, index, and log.
- Subject and scope are awaiting confirmation; no source material has been ingested.

## [2026-09-29] ingest | KV cache surveys and systems papers

- Ingested the owner-provided `raw/survey_on_kv.pdf` and `raw/from_tensor_buffer.pdf` into [source notes](sources/).
- Added seven open-access systems papers under `raw/papers/`: PagedAttention (SOSP 2023), DistServe (OSDI 2024), InfiniGen (OSDI 2024), CacheGen (SIGCOMM 2024), vAttention (ASPLOS 2025), Mooncake (FAST 2025), and FlashInfer (MLSys 2025). Their source notes record the canonical publication and downloaded version.
- Created [the overview](overview.md) and [KV cache management topic page](topics/kv-cache-management.md), and updated [the index](index.md).
- Preserved the original PDFs in `raw/` without modification. Three other PDFs already present there—Quest, RetrievalAttention, and RoarGraph—remain queued for a later ingest.

## [2026-09-29] setup | Initialize Git repository

- Initialized the workspace on branch `main`; no commit has been created.

## [2026-09-29] ingest | Sparse attention, ANNS, and CXL systems

- Ingested the five previously unprocessed PDFs: `raw/deepseek_v3.2.pdf`, `raw/quest.pdf`, `raw/retrievalatten.pdf`, `raw/retroinfer.pdf`, and `raw/roargraph.pdf`.
- Added two open-access systems papers to `raw/papers/` and ingested them: CXL-ANNS (USENIX ATC 2023) and ECHO (OSDI 2026).
- Created seven source notes: [DeepSeek-V3.2](sources/deepseek-v3-2.md), [Quest](sources/quest.md), [RetrievalAttention](sources/retrievalattention.md), [RoarGraph](sources/roargraph.md), [RetroInfer](sources/retroinfer.md), [CXL-ANNS](sources/cxl-anns.md), and [ECHO](sources/echo.md).
- Added [the sparse attention/CXL/ANNS synthesis](topics/sparse-attention-cxl-anns.md), linked it from the existing [KV cache topic](topics/kv-cache-management.md), refreshed the [overview](overview.md) and [index](index.md), and updated the supplemental-paper list.
- The seven-paper set connects sparse token selection, OOD-aware ANNS, CXL graph search, and host-tier KV serving; none of these papers evaluates all three of sparse attention, CXL, and ANNS in one system.

## [2026-09-29] ingest | CXL-backed sparse KV systems

- Added and ingested [Beluga](sources/beluga.md) (SIGMOD 2026) and [SAC](sources/sac.md) (arXiv v1, 2026) into `raw/papers/`.
- Updated the [sparse attention/CXL/ANNS synthesis](topics/sparse-attention-cxl-anns.md), overview, index, and supplemental-paper list to distinguish three pairwise paths: RetrievalAttention (sparse attention + ANNS), SAC (sparse attention + CXL), and CXL-ANNS (CXL + ANNS).
- Recorded the evidence boundary: SAC uses DeepSeek's top-k indexer rather than ANNS; Beluga's CXL/KV serving paper does not integrate ANNS, though it discusses vector/graph databases as a possible CXL use.

## [2026-09-29] organize | Consolidate ingested papers

- Moved the seven already-ingested root-level PDFs into `raw/papers/`, preserving their filenames and contents.
- Updated source-note links and completed [`raw/papers/README.md`](../raw/papers/README.md) as the catalog of all 18 ingested papers.
- Replaced a stale Quest cross-reference in the InfiniGen source note with the existing Quest note.

## [2026-09-29] ingest | Broaden CXL, sparse-attention, and physical-planning related work

- Ingested 20 additional external papers and the owner-provided [CXL-Vector anonymized SIGMOD ’27 submission manuscript](sources/cxl-vector.md); the corpus now contains 39 PDFs, each with a source note.
- Added direct serving and execution comparisons: [IceCache](sources/icecache.md), [SPIN](sources/spin.md), [Fluxion](sources/fluxion.md), [HiSparse](sources/hisparse.md), [ScoutAttention](sources/scoutattention.md), [CompactAttention](sources/compactattention.md), [LayerKV](sources/layerkv.md), [Strata](sources/strata.md), [DirectKV](sources/directkv.md), and [SWARM](sources/swarm.md).
- Added selector/compression and index-maintenance references: [MiniMax Sparse Attention](sources/minimax-sparse-attention.md), [Louver](sources/louver.md), [Verified vAttention](sources/vattention-verified.md), [Self-Indexing KVCache](sources/self-indexing-kvcache.md), [SALS](sources/sals.md), and [FreshDiskANN](sources/freshdiskann.md).
- Added CXL data-path and ANN comparisons: [Exploring CXL KV Storage](sources/exploring-cxl-kv-storage.md), [COSMOS](sources/cosmos-cxl-anns.md), [PNM-KV](sources/pnm-kv.md), and [TRACE](sources/trace-cxl.md).
- Moved all new PDFs into `raw/papers/`, preserving the supplied CXL-Vector filename and contents, and appended all 21 entries to [the paper catalog](../raw/papers/README.md).
- Updated the [index](index.md), [overview](overview.md), [KV cache topic](topics/kv-cache-management.md), and [sparse attention/CXL/ANNS synthesis](topics/sparse-attention-cxl-anns.md). The refreshed synthesis narrows the proposal hypothesis to fixed-selection, physical-footprint-aware routing within a KV group; this remains an experimental question, not an established novelty claim.
