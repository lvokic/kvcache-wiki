# Unifying Sparse Attention with Hierarchical Memory for Scalable Long-Context LLM Serving

**作者：** Zihan Zhao 等。**版本：** arXiv:2604.26837 v1，2026 预印本。
**本地原件：** [spin-arxiv26.pdf](../../raw/papers/spin-arxiv26.pdf)
**来源：** [arXiv:2604.26837](https://arxiv.org/abs/2604.26837)

## 摘要与设计

SPIN 将多种稀疏粒度接入共同的 page-based KV substrate：算法提供 Index/Select 逻辑，其余执行由统一 pipeline 承担。系统再根据 locality 动态设定 request 的 HBM budget，采用 GPU-friendly bucketed LRU，并用按 active working set 组织的两层 metadata 控制规模。论文报告，在 vLLM 上集成三类 sparse attention 后，端到端吞吐相对原 vLLM 提高 1.66–5.66 倍、TTFT 降低 7–9 倍；这些数值对应作者的 workload 和 GPU 配置。

## 与当前 proposal 的边界

- SPIN 已覆盖稀疏粒度统一、分页存储、动态 HBM 管理和 metadata 扩展；“提供统一 KV 管理接口”不宜作为独立创新点。
- 论文的主要路径是 GPU/CPU 间的 cache residency 和搬运。它没有验证 CXL 上的 attention-aware ANNS，也不是固定逐 head 选集下按真实 span、命中、GQA 重叠拆分同一 group 的 CPU/GPU partial-attention planner。
- 这是 arXiv 预印本；其系统结果应按该版本的实验边界解读。

## 相关页面

- 与 [HiSparse](hisparse.md) 对照 exact selection resolve，与 [Fluxion](fluxion.md) 对照异构 attention 执行。
- 参见 [KV cache 管理与 serving 总览](../topics/kv-cache-management.md)。
