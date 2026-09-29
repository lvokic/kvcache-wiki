# Self-Indexing KVCache: Predicting Sparse Attention from Compressed Keys

**作者：** Xu Yang, Jiapeng Zhang, Dongyang Zhao, Guo Chen, Zhuo Tang。**发表：** AAAI 2026, 40(33), 27675–27683。
**本地原件：** [self-indexing-kvcache-aaai26.pdf](../../raw/papers/self-indexing-kvcache-aaai26.pdf)
**正式来源：** [AAAI 论文页与 DOI 10.1609/aaai.v40i33.39988](https://ojs.aaai.org/index.php/AAAI/article/view/39988) · [arXiv:2603.14224](https://arxiv.org/abs/2603.14224)

## 摘要与设计

本文将压缩 key 表示同时用作 sparse token selector：以 sign-based 1-bit vector quantization 为 key 子向量分组，再用轻量 codebook/LUT-GEMV 检索候选；自定义 CUDA kernel 将检索与 attention 计算集成。作者报告可降低 KV memory footprint 并维持其测试任务的质量。

## 边界与联系

- 它说明压缩 representation 本身可承担 index 功能，因此 proposal 的 SQ8/TQ5 导航需要比较“独立 compact code/index”与“压缩 KV 即索引”的成本和检索质量。
- sign code 的排序能力依赖其表示假设；这不是对 attention 原始点积排序的普遍等价保证。论文没有研究 CXL placement、OOD graph traversal 或多设备 partial attention。
- 不要与本仓库原有 [vAttention](vattention.md) 混淆；其主题不同。

## 相关页面

- 参见 [SALS](sals.md)、[CXL-Vector](cxl-vector.md) 和 [综合页](../topics/sparse-attention-cxl-anns.md)。
