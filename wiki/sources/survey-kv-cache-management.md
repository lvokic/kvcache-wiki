# A Survey on Large Language Model Acceleration based on KV Cache Management

**作者：** Haoyang Li, Yiming Li, Anxin Tian, Tianhao Tang, Zhanchao Xu, Xuejia Chen, Nicole Hu, Wei Dong, Qing Li, Lei Chen  
**发表：** TMLR 2025；本地文件为 arXiv v3（2025-07-30）  
**原始文件：** [raw/survey_on_kv.pdf](../../raw/survey_on_kv.pdf)  
**正式记录：** [arXiv:2412.19442](https://arxiv.org/abs/2412.19442) · [TMLR review](https://openreview.net/pdf?id=z3JZzu9EA3)

## 摘要

这篇综述按优化发生的位置整理 KV cache 工作：token-level（选择、预算分配、合并、量化、低秩分解）、model-level（注意力分组/共享及架构调整）和 system-level（内存管理、调度、硬件感知设计）。它也汇总文本与多模态任务的数据集和评测指标。详见原文 §III–VII。

## 主要价值

- 作为领域地图，它把“减少、压缩或复用哪些 KV”与“如何在 serving 系统中管理 KV”放进同一个框架。
- §VI 将系统工作分为内存管理、调度和硬件感知设计，并继续整理 prefix-aware、抢占/公平性、分层调度、I/O 与异构硬件等方向。
- 适合作为入门目录和术语索引；本文的广度比具体系统实现分析更有用。

## 范围与注意事项

- 这是截至 2025-07 修订的综述。之后发表的系统不在这个版本的覆盖范围内。
- 这里的 system-level 是较宽的类别。后来的系统综述指出，这种按优化层次分类会把设计差异很大的系统放入同一类，例如 DistServe 与 KVDirect 的 KV ownership 选择不同。两种 taxonomy 回答的问题不同：本文主要回答“优化发生在哪一层”，见 [From Tensor Buffer to Distributed Memory Hierarchy](distributed-kv-hierarchy-survey.md) §7.1。
- 本文的分类和 benchmark 总结不是统一硬件、工作负载和 SLO 下的横向性能排名。引用其中的量化结果时，应回到对应原论文核对实验配置。

## 与本 wiki 的连接

- 与 [KV cache 管理与 serving 系统](../topics/kv-cache-management.md) 的三层 taxonomy 对应。
- 和 [From Tensor Buffer to Distributed Memory Hierarchy](distributed-kv-hierarchy-survey.md) 互补：后者更细看系统的 locality、lifetime、ownership 和 substrate。

