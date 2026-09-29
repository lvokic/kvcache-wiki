# vAttention: Verified Sparse Attention via Sampling

**作者：** Aditya Desai, Kumar Krishna Agrawal, Shuo Yang 等。**发表：** ICLR 2026。
**本地原件：** [vattention-verified-iclr26.pdf](../../raw/papers/vattention-verified-iclr26.pdf)
**来源：** [ICLR 2026 proceedings](https://proceedings.iclr.cc/paper_files/paper/2026/hash/55cb562b1f5af71f6707f3ff3c7941e6-Abstract-Conference.html) · [arXiv:2510.05688](https://arxiv.org/abs/2510.05688)

## 摘要与设计

本文将 top-k 和随机采样结合：当注意力集中时使用 top-k，当 score 较分散时随机采样估计，并为近似质量提供用户指定的 epsilon/delta 保证。作者报告在多个 benchmark 上改善 sparse-attention 质量，并在其测试中以较高稀疏度接近 full-model 质量。

## 与本研究的关系

- 它为固定选集 planner 研究提供 selector/attention-approximation 质量的参照。必须区分 sparse-vs-full 的算法误差与相同 sparse mask 在 CPU/GPU/CXL 上执行的浮点归约误差。
- 保证针对论文定义的估计/误差模型；不能自动解释为任意 downstream task 的质量保证，也没有 CXL、ANNS 或分层 KV placement 评估。
- 该论文和仓库原有 [vAttention](vattention.md) 不同：旧文是 CUDA VMM 动态 KV allocation；本文是带近似保证的 sparse attention 方法。

## 相关页面

- 参见 [Louver](louver.md)、[MiniMax Sparse Attention](minimax-sparse-attention.md) 和 [综合页](../topics/sparse-attention-cxl-anns.md)。
