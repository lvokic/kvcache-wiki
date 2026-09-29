# FlashInfer: Efficient and Customizable Attention Engine for LLM Inference Serving

**作者：** Zihao Ye, Lequn Chen, Ruihang Lai, Wuwei Lin, Yineng Zhang, Stephanie Wang, Tianqi Chen, Baris Kasikci, Vinod Grover, Arvind Krishnamurthy, Luis Ceze  
**发表：** MLSys 2025  
**本地 PDF：** [raw/papers/flashinfer-mlsys25.pdf](../../raw/papers/flashinfer-mlsys25.pdf)，MLSys 正式论文版  
**出处：** [MLSys 2025 论文 PDF](https://proceedings.mlsys.org/paper_files/paper/2025/file/dbf02b21d77409a2db30e56866a8ab3a-Paper-Conference.pdf) · [arXiv:2501.01005](https://arxiv.org/abs/2501.01005)

## 摘要

FlashInfer 提供可定制的 LLM attention engine。它用 block-sparse/composable formats 统一不同 KV layout，以 JIT template 适配 attention 变体，并通过 dynamic-aware runtime scheduler 处理批次中变化的 query 和 KV 长度，同时兼容 CUDAGraph。论文描述其已集成到 SGLang、vLLM 和 MLC-Engine。见原文 §3–4。

## 关键价值与证据

- 它补上了 cache manager 与 GPU attention kernel 之间的接口问题：KV 的分页、稀疏布局和变长批处理会怎样影响 kernel 数据读取、调度和负载均衡。
- 论文报告相较 compiler backends 的 serving benchmark 上 ITL 降低 29%–69%；长上下文推理延迟降低 28%–30%；并行生成 speedup 为 13%–17%。这些是论文指定的比较基线和场景，不是统一硬件下相对所有 serving engines 的保证。出处：[MLSys 正式论文 §5](https://proceedings.mlsys.org/paper_files/paper/2025/file/dbf02b21d77409a2db30e56866a8ab3a-Paper-Conference.pdf)。

## 边界与连接

FlashInfer 是 KV 的计算数据路径和 attention-engine 抽象，不负责决定 KV 在多长时间内保留、由谁拥有或何时从 GPU 换出。与 [PagedAttention](pagedattention.md) 和 [vAttention](vattention.md) 对照时，应把 allocator/地址空间策略与 kernel layout/runtime 分开看。

