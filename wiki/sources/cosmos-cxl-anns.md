# COSMOS: A CXL-Based Full In-Memory System for Approximate Nearest Neighbor Search

**作者：** Seoyoung Ko 等。**发表：** IEEE Computer Architecture Letters, 24(1), 173–176 (2025)；arXiv 版本 v1。
**本地原件：** [cosmos-cxl-anns-ieee-cal25.pdf](../../raw/papers/cosmos-cxl-anns-ieee-cal25.pdf)
**来源：** [arXiv:2505.16096](https://arxiv.org/abs/2505.16096) · [DOI 10.1109/LCA.2025.3570235](https://doi.org/10.1109/LCA.2025.3570235)

## 摘要与设计

COSMOS 面向 billion-scale RAG vector search，将 general-purpose cores 集成到 CXL memory devices 中执行 ANNS，并提出 rank-level parallel distance computation 和 adjacency-aware 跨 CXL device placement。SIFT1B、DEEP1B 评估报告相对论文所设 CXL baseline 最高 6.72 倍吞吐、相对所测 CXL solution 2.35 倍。

## 与 CXL-Vector 的对照

- COSMOS 把计算移到 CXL devices 附近；用户的 CXL-Vector 稿件针对 commodity memory-only CXL，主要由主机端 SIMD 批量导航。因此两者的 CXL endpoint、数据路径和硬件假设不同。
- COSMOS 是通用 RAG vector ANNS，不处理 attention Q/K OOD、KV 生命周期或 sparse-attention 质量。
- 它扩大了 CXL+ANN 的架构设计空间，也提示 proposal 需声明 memory-only 限定，并将 endpoint near-data processing 作为另一个体系结构方向而非可直接复用的 baseline。

## 相关页面

- 参见 [CXL-ANNS](cxl-anns.md)、[CXL-Vector](cxl-vector.md)、[PNM-KV](pnm-kv.md) 和 [综合页](../topics/sparse-attention-cxl-anns.md)。
