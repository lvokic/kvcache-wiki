# Sparse Attention as a Range Searching Problem: Towards an Inference-Efficient Index for KV Cache

**作者：** Mohsen Dehghankar, Abolfazl Asudeh。**版本：** arXiv:2605.06763，2026 预印本。
**本地原件：** [louver-arxiv26.pdf](../../raw/papers/louver-arxiv26.pdf)
**来源：** [arXiv:2605.06763](https://arxiv.org/abs/2605.06763)

## 摘要与设计

Louver 将 sparse attention 的候选检索表述为 halfspace range searching，构建面向 KV retrieval 的索引，并针对 CPU/GPU 实现优化。它对设定阈值以上的“相关 keys”提供零 false negative 保证，目标是避免漏掉阈值定义的重要项。

## 解释边界

- 这是 KV sparse index 与质量保证的直接相关工作，应该纳入 selector/ANNS 对照。
- 零 false negative 是相对于给定阈值集合的保证；它本身不代表精确 top-k、保留固定数量 tokens、attention 输出误差上界或 full-attention 等价。阈值召回量和索引成本也可能随 query 变化。
- 文稿报告的查询速度和质量来自其定义的问题、硬件与对比方法；不等同于 CXL 上远端 ANN 或 physical planner 的性能。

## 相关页面

- 与 [RetrievalAttention](retrievalattention.md)、[RoarGraph](roargraph.md) 和 [Verified Sparse Attention](vattention-verified.md) 对照；参见 [综合页](../topics/sparse-attention-cxl-anns.md)。
