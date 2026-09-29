# InfiniGen: Efficient Generative Inference of Large Language Models with Dynamic KV Cache Management

**作者：** Wonbeom Lee, Jungi Lee, Junghwan Seo, Jaewoong Sim  
**发表：** OSDI 2024, pp. 155–172  
**本地 PDF：** [raw/papers/infinigen-osdi24.pdf](../../raw/papers/infinigen-osdi24.pdf)，USENIX 正式论文版  
**出处：** [USENIX OSDI 2024](https://www.usenix.org/conference/osdi24/presentation/lee) · [论文 PDF](https://www.usenix.org/system/files/osdi24-lee.pdf) · [arXiv:2406.19707](https://arxiv.org/abs/2406.19707)

## 摘要

InfiniGen 面向 host-memory offload 的长文本生成系统。它用当前层输入、下一层部分 query 权重和部分 key cache，对下一层可能关注的 token 做轻量预测，只把选中的 KV entries 预取到 GPU；其余 KV 留在 CPU memory，并动态移除低频 token。见原文 §3–4。

## 关键价值与证据

- 把 query-aware 选择变成面向 offload 的系统机制：重点不只是少算 attention，而是减少每步从 host memory 搬到 GPU 的数据量。
- 论文在其 offloading baseline 和代表性模型上的评测报告最高 3.00× 性能提升，并声称比已有 KV 管理方法有更好的准确率。该数字是特定 baseline、模型和 workload 下的上限，不应跨论文直接比较。出处：[USENIX 摘要与论文](https://www.usenix.org/conference/osdi24/presentation/lee)。

## 边界与连接

- 设计依赖 offloading 架构和预测阶段额外工作；它不是一个对所有 serving 系统透明的通用 cache manager。
- 它与 [PagedAttention](pagedattention.md) 关注不同层次：PagedAttention 管理 GPU 物理 blocks；InfiniGen 决定哪些 host-resident KV 要预取、哪些低使用项可淘汰。
- 与 [Quest](../../raw/quest.pdf) 等 query-aware 稀疏注意力方向相邻，但本轮尚未为 `quest.pdf` 建立 source note。

