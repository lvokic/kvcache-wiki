# Efficient Memory Management for Large Language Model Serving with PagedAttention

**作者：** Woosuk Kwon, Zhuohan Li, Siyuan Zhuang, Ying Sheng, Lianmin Zheng, Cody Hao Yu, Joseph E. Gonzalez, Hao Zhang, Ion Stoica  
**发表：** SOSP 2023, pp. 611–626  
**本地 PDF：** [raw/papers/pagedattention-sosp23.pdf](../../raw/papers/pagedattention-sosp23.pdf)，取自 arXiv v1  
**出处：** [ACM DOI 10.1145/3600006.3613165](https://doi.org/10.1145/3600006.3613165) · [arXiv:2309.06180](https://arxiv.org/abs/2309.06180)

## 摘要

PagedAttention 把 OS paging 的逻辑/物理地址分离思想用于 KV cache：请求的逻辑 token blocks 映射到 GPU 上不必连续的物理 blocks，block table 在生成时按需扩展。vLLM 在此基础上实现 KV block 管理、请求调度与共享。见原文 §4。

## 关键设计与证据

- 按固定 token 数划分 KV blocks，避免先按最大生成长度预留一整块连续空间；每个请求最后一个 block 的内部空位是主要剩余碎片来源。见 §4.1–4.2。
- 对并行采样、beam search 和共享前缀，vLLM 用引用计数共享 blocks，并在写时复制。论文在其测试中报告 beam search 下显著的内存节省和吞吐提升；具体结果依赖模型、trace 和 block size。见 §4.4、§6.3–6.4。
- 论文也量化了 paged layout 的代价：其 attention kernel 在微基准中比 FasterTransformer 高 20–26% 延迟，原因包括 block-table 访问、分支和变长处理；论文报告端到端收益仍为正。block 过大增加内部碎片，过小则降低并行利用率。见 §7.1–7.2。

## 重要边界

这项工作建立的是 GPU 本地 KV 的分页管理基线。它没有解决跨节点长期保留或全局缓存 ownership；后续的 [DistServe](distserve.md) 与 [Mooncake](mooncake.md) 把问题扩展到网络传输和跨请求复用。数值结果是论文自有实验，不可直接与不同年代硬件上的论文数字比较。

