# From Tensor Buffer to Distributed Memory Hierarchy: A Survey of KV Cache Management for LLM Serving

**作者：** Jie Li, Tongyang Wang, Yong Chen  
**版本：** arXiv v1，2026-06-30；截至本次整理尚未标注同行评审发表 venue  
**原始文件：** [raw/papers/from_tensor_buffer.pdf](../../raw/papers/from_tensor_buffer.pdf)  
**正式记录：** [arXiv:2607.02574](https://arxiv.org/abs/2607.02574) · [HTML 全文](https://arxiv.org/html/2607.02574)

## 摘要

这篇系统综述把 KV cache 从单个请求的 GPU tensor 重新看作 serving 系统中的显式内存对象。它以四个维度描述系统：**locality**（状态位于何处）、**lifetime**（保留多久）、**ownership**（谁命名、分配、淘汰并仲裁访问）和 **substrate**（数据通过什么内存或传输路径移动）。作者据此归纳五类架构：local-paged、disaggregated-pipeline、shared-store、memory-pool 和 hybrid-tier。详见原文 §3–4。

## 值得带走的观察

- Prefill 和 decode 对资源的需求不同：prefill 偏计算吞吐；decode 反复读取增长中的 KV，更受 KV 带宽、容量和远程访问延迟影响。KV placement 因此和 TTFT、TPOT、goodput 一起设计，而不是独立的存储选择。见 §2.1–2.2。
- **Ownership 是独立的设计选择。** 相同 substrate 可以由每个 worker、本地/全局 coordinator、分布式 directory，或 shared-memory convention 管理；这会改变控制开销、协调复杂度和故障语义。见 §3.3。
- 架构不是简单的替代序列。local paging、phase disaggregation、shared store、memory pool 与 tiered design 仍同时存在；复杂系统会组合多个 archetype。Mooncake 被归为 hybrid-tier，而它的 store 子系统又具有 shared-store 特征。见 §4.3–4.5。
- 作者审计了 35 个系统/框架条目（33 个不同系统或模式），并指出论文间缺少可比的远程 KV 访问模式、metadata path 成本、操作完成开销、reuse distance、prefetchability、P99 延迟归因和公开 trace。见 §5.1、§5.5。

## 可检验的研究问题

作者提出一个**待检验预测**：在工作负载和吞吐匹配后，集中式 ownership 的尾延迟方差可能低于分布式 ownership，因为后者要承担 directory、锁或协调开销；文中把 DistServe 与 KVDirect 作为候选对照。作者同时指出，现有论文没有足够的 P99 分解来验证这一预测。应把它当作研究假设，而不是已证实结论。见 §5.4。

## 范围与注意事项

- 本文是 arXiv v1 的选择性系统 taxonomy，检索截止到 2026-05。分类是作者的分析框架，不是经过统一测量得出的排名。
- 作者强调不同论文使用不同模型、硬件、上下文分布和 SLO，不能根据单个 throughput 或 latency 数字直接排出系统优劣。公开生产 trace 和细粒度归因指标仍不足。见 §5.2、§5.5。
- 它对 ownership、lifetime 和 substrate 的分析与 [2025 年 KV cache survey](survey-kv-cache-management.md) 的 token/model/system 优化分类互补，不应混为同一维度。

## 与本 wiki 的连接

- 本 wiki 的主综合页：[KV cache 管理与 serving 系统](../topics/kv-cache-management.md)。
- 对照 [DistServe](distserve.md)（请求级 P/D handoff）、[Mooncake](mooncake.md)（全局复用与多层存储）和 [CacheGen](cachegen.md)（传输时压缩）。
