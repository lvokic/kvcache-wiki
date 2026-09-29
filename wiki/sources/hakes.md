# HAKES: Scalable Vector Database for Embedding Search Service

## Bibliographic details

- Authors: Guoyu Hu, Shaofeng Cai, Tien Tuan Anh Dinh, Zhongle Xie, Cong Yue, Gang Chen, and Beng Chin Ooi.
- Venue: PVLDB 18(9), 3049–3062, 2025. DOI: [10.14778/3746405.3746427](https://doi.org/10.14778/3746405.3746427).
- Archived PDF: [`raw/papers/hakes-arxiv25.pdf`](../../raw/papers/hakes-arxiv25.pdf), arXiv:2505.12524. The repository stores the public arXiv version; bibliographic details refer to the PVLDB publication.
- Canonical paper: [PVLDB PDF](https://www.vldb.org/pvldb/vol18/p3049-ooi.pdf); [artifact repository](https://github.com/nusdbsystem/HAKES-Search).

## Summary

HAKES targets high-recall vector search while inserts and queries run concurrently. Its HAKES-Index is a partitioning-based ANN index that combines dimensionality reduction, IVF coarse partitions, product-quantized vectors for candidate filtering, and full-vector reranking. It separates the parameters used to encode arriving vectors from those used to search, allowing new vectors to be encoded and appended without retraining or replacing the active search parameters on every insert.

## Key claims from the paper

- **Index structure (§3.1):** IVF centroids assign inserted vectors to partitions and rank partitions at query time. Compressed vectors are scanned in the filter stage; candidate vectors are reranked using full-precision data.
- **Concurrent inserts (§3.1, §5.3):** an insert transforms and quantizes a vector using the insert-side parameters, then appends it to its assigned partition and the full-vector buffer. The authors evaluate concurrent read-write operation with 32 clients and vary the write ratio; Figure 10 reports the comparison at recall 0.99. Their FAISS extension also adds partition locking to IVFPQ_RF and OPQIVFPQ_RF baselines.
- **Reported outcome (§5.3):** partitioning-based indexes have lower contention and more predictable memory access than the tested HNSW baseline. HAKES-Index is the paper’s strongest proposed index in its evaluations; these results are tied to the tested data, parameters, implementation, and workloads.
- **Deletion (§3.1):** deletions use tombstones, so this path avoids immediate list compaction but does not demonstrate a full solution to long-term physical reclamation.

## Limitations and evidence boundary

The paper evaluates embedding-search datasets and its own FAISS extension, not CXL-attached memory. Its insert/search parameter split reduces coordination around learned parameters, but the paper does not establish that fixed insert-side centroids remain balanced under arbitrary distribution drift. The tested concurrent write ratios and client count are useful baseline settings, not a universal production workload. The deletion path is tombstone-based.

## Relevance to this wiki

HAKES is the closest CPU-side precedent in this collection for IVF-style concurrent query and insert. It should be a baseline for CXL-Vector follow-on work, while preserving the distinction between its compressed filter-and-refine index and plain IVFFlat/IVFPQ. See [Dynamic IVF and concurrent read-write](../analysis/concurrent-ivf-read-write.md) and [Sparse attention, CXL, and ANNS](../topics/sparse-attention-cxl-anns.md).
