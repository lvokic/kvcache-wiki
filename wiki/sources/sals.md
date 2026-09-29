# SALS: Sparse Attention in Latent Space for KV Cache Compression

**作者：** Junlin Mu, Hantao Huang, Jihang Zhang, Minghui Yu, Tao Wang, Yidong Li。**版本：** arXiv:2510.24273，2025 预印本。
**本地原件：** [sals-arxiv25.pdf](../../raw/papers/sals-arxiv25.pdf)
**来源：** [arXiv:2510.24273](https://arxiv.org/abs/2510.24273)

## 摘要与设计

SALS 用低秩投影把 KV cache 映射到 latent space，并以 RoPE-free query/key interaction 在该空间筛选 tokens；只对选中项重建表示后执行 sparse attention。作者的动机是 RoPE 会改变 key 的低秩特性，直接在 RoPE 后低秩表示上检索可能带来质量或重建成本问题。论文在 LLaMA2-7B、Mistral-7B 和 RULER 128K 设置上评估，报告不同配置下的压缩、attention kernel 和端到端收益。

## 对本 proposal 的意义

- 它是“压缩 K 如何参与检索、是否需要恢复原始 K”的相关算法基线，可用于比较低秩 latent selection 与 CXL-Vector 式 SQ8/TQ5 图导航。
- SALS 不是 CXL 系统，也不使用图 ANNS；其稀疏选择和重建路径不能直接证明 attention OOD graph 的质量或 CXL 物理规划收益。
- 该版本标注 source code future release；本文报告的设置与不同系统硬件上的结果不可直接排名。

## 相关页面

- 参见 [Self-Indexing KVCache](self-indexing-kvcache.md)、[RetrievalAttention](retrievalattention.md) 与 [综合页](../topics/sparse-attention-cxl-anns.md)。
