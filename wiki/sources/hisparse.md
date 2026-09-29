# HiSparse: Scaling Sparse-Attention Decoding with Hierarchical KV Cache Management

**作者：** Zhiqiang Xie, Zhangheng Huang, Tingwei Huang, Ziyi Xu, Ruiyang Ma, Christos Kozyrakis。**版本：** arXiv:2608.07009 v1，2026-08 预印本。
**本地原件：** [hisparse-arxiv26.pdf](../../raw/papers/hisparse-arxiv26.pdf)
**来源：** [arXiv:2608.07009](https://arxiv.org/abs/2608.07009)

## 摘要与设计

HiSparse 是 indexer-agnostic 的 exact hierarchical KV cache：全量 KV 留在 host memory，GPU 只保留固定大小 cache。融合 CUDA kernel 在 decode CUDA graph 中解析各层 sparse selection 的 cache hit、LRU replacement 与 host-to-device fetch；当模型跨层共享 selection 时，按层精确预取以隐藏 miss 延迟。论文称只改变 KV placement，因此模型输出保持不变；在 SGLang 中集成 DSA、NSA 和 Quest，并在 H200、B200、GH200 上报告最高 4.7 倍 peak generation throughput。

## 对 proposal 的影响与边界

- 这是本 proposal 最强的新对照之一：exact selection resolution、受限 GPU residency、层间预取和 CUDA-graph 内管理均已有系统实现。仅以“只在 sparse selector 命中处把 KV 搬到 GPU”作为贡献不足。
- HiSparse 的 host miss 路径用于 cache placement；它没有使用 CXL，也没有 CPU partial attention 或按 group 冷 footprint 把部分选集留在 CPU 计算。因此需测同一 selection 下，GPU fetch/cache 路径与 CPU partial + bounded packet 的全成本差异。
- 论文报告“comparable per-token latency”而非零 I/O 成本；应分别记录 IO、resolve 与 decode 负担。

## 相关页面

- 与 [SPIN](spin.md)、[Fluxion](fluxion.md) 和 [SAC](sac.md) 对照；参见 [综合页](../topics/sparse-attention-cxl-anns.md)。
