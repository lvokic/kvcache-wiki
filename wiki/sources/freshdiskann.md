# FreshDiskANN: A Fast and Accurate Graph-Based ANN Index for Streaming Similarity Search

**作者：** Aditi Singh, Suhas Jayaram Subramanya, Ravishankar Krishnaswamy, Harsha Vardhan Simhadri。**版本：** arXiv:2105.09613，2021 预印本。
**本地原件：** [freshdiskann-arxiv21.pdf](../../raw/papers/freshdiskann-arxiv21.pdf)
**来源：** [arXiv:2105.09613](https://arxiv.org/abs/2105.09613)

## 摘要与设计

FreshDiskANN 研究支持并发插入、删除和搜索的 graph ANN index。其 FreshVamana 更新算法尽量以与增量规模相关的工作维护搜索质量；StreamingMerge 将更新合并到 SSD-resident index 并控制写放大。论文展示的是通用 streaming similarity search，不是每 token 追加的 attention serving。

## 与本 proposal 的关系

- 它为 proposal 的 append、delta、sealed segment、重建/compaction 讨论提供动态 ANN 先例；不能把“ANN 图需要追加协议”描述成没有相关工作。
- 其 storage medium、更新速率和毫秒级 ANN 目标与在线 KV 每层每步检索不同；具体更新成本、snapshot publication 和 query tail 都需在 attention workload 重测。
- 这是 arXiv 预印本；未将其标成系统会议正式发表论文。

## 相关页面

- 参见 [RoarGraph](roargraph.md)、[RetrievalAttention](retrievalattention.md) 和 [综合页](../topics/sparse-attention-cxl-anns.md)。
