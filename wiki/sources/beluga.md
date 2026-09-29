# Beluga: A CXL-Based Memory Architecture for Scalable and Efficient LLM KVCache Management

**作者：** Xinjun Yang 等  
**发表：** ACM SIGMOD 2026，Proc. ACM Manag. Data 4(1)，Article 13；DOI 10.1145/3786627  
**本地版本：** [raw/papers/beluga-sigmod26.pdf](../../raw/papers/beluga-sigmod26.pdf)，arXiv v2  
**正式记录：** [ACM DOI](https://doi.org/10.1145/3786627) · [arXiv 全文](https://arxiv.org/abs/2511.20172)

## 摘要

Beluga 以一台商用 CXL 2.0 switch 建立多主机共享 memory pool，并把它接入 LLM KV cache 管理。与 RDMA pool 相比，CXL 的 load/store 接口减少消息协议、bounce buffer 和完成队列等数据/控制路径成本。系统文章不以 sparse attention 或 ANNS 为主要目标，但为“KV payload 在 CXL，索引/查询在另一处”的设计提供实测内存路径证据。

## 设计与结果

- Beluga 基于 XConn XC50256 CXL switch，设计多主机共享池、GPU/CPU 映射、CXL coherence 管理与数据布局。作者指出 PCIe/CXL adapter 和 root-complex 路径仍影响 GPU–CXL 带宽；直连 GPU 到 switch 被列为未来方向。见 §4–5。
- 在 cache-hit 设置中，Beluga-KVCache 相较其 RDMA-based baseline 报告平均 TTFT 降低 89.6%、QPS 提高 7.35×。这些是论文所测平台和 cache-hit 负载的结果。见 §6。
- 作者还测量从 CXL/RDMA 取回 16 个非连续 sparse KV tokens 的 microbenchmark：Qwen-32B 设置中，CXL latency 比 RDMA 低 95.9%。这是细粒度传输 microbenchmark，不是 end-to-end sparse-attention 服务结果。见 §7、Fig. 14–15。
- 讨论部分把 CXL memory pool 扩展到 vector/graph databases（如 HNSW）列为潜在应用；该论文没有实际实现或评测 ANNS index。

## 边界与限制

- main KVCache serving path 对比的是 CXL 和 RDMA memory pools；它没有集成 DeepSeek/Quest 的动态 sparse selector，也没有评估 ANNS graph traversal。
- 实验基于具体商用 CXL 2.0 switch 与特定多主机拓扑；GPU–CXL 数据路径、共享池 coherence 和带宽扩展仍有明确的硬件/系统边界。
- 论文中的 7.35× 与 89.6% 来自 cache-hit workload；不能外推到所有 cache-miss 比例、稀疏度或 CXL fabric。

## 与本 wiki 的连接

- [SAC](sac.md) 把关注点推进到 sparse attention 时按需从 CXL 取回 top-k KV；它仍使用模型的 DSA indexer，不是 ANNS。
- [CXL-ANNS](cxl-anns.md) 展示 CXL 上通用 ANN graph 的缓存、预取与近数据计算；[RetrievalAttention](retrievalattention.md) 展示 attention-aware query/key 图索引。
- 参见 [sparse attention、CXL 与 ANNS 综合页](../topics/sparse-attention-cxl-anns.md)。
