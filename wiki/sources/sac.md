# SAC: Disaggregated KV Cache System for Sparse Attention LLMs with CXL

**作者：** Ruiyang Ma, Teng Ma, Junru Li, Hantian Zha, Xuchun Shang, Qingda Hu, Zheng Liu, Xinjun Yang, Tao Ma, Guojie Luo  
**版本：** arXiv:2606.19746 v1，2026-06-18 预印本  
**本地原件：** [raw/papers/sac-cxl-sparse-arxiv26.pdf](../../raw/papers/sac-cxl-sparse-arxiv26.pdf)  
**正式记录：** [arXiv 摘要](https://arxiv.org/abs/2606.19746) · [HTML 全文](https://arxiv.org/html/2606.19746)

## 摘要

SAC（Sparse Attention on CXL）是本 wiki 里最直接的 sparse-attention + CXL 系统案例。它把完整 KV cache 保存在共享 CXL pool；在每层 attention 时，根据现有 sparse selector 产生的 top-k indices，从 CXL 按需读取非连续 KV entries 到 GPU。CXL 同时管理 KV data 与共享元数据。它没有使用 ANNS 图索引：token selection 仍由 DeepSeek-V3.2 的 DSA indexer 完成。

## 设计与结果

- SAC 集成 HiSparse/SGLang；indexer keys 留在 GPU，DSA 为每个 query 选出 top-k（论文配置 2048）KV latent vectors；KV payload 留在 CXL pool，GPU 使用 coalesced vector loads 读取，生成的新 KV 写回 CXL。见 §2.1、§4.1–4.3。
- 架构使用 PCIe 5.0/CXL switch 和多台主机可共享的全局地址空间；论文所列测试平台为 8×H20、2TB CXL pool。见 §4.2、Appendix A。
- DeepSeek-V3.2、16K–128K context 的 Round-2 decode 测试中，相较 RDMA full-prefetch baseline，报告平均 throughput 高 2.1×、TTFT 低 9.7×、TBT 低 1.8×；相较 local DRAM upper bound，throughput 约低 9%。这些结果仅对应作者的系统、配置与 workload。见 §5.1–5.3。

## 边界与限制

- 该系统解决 sparse selection 之后的细粒度 KV payload 访问，并未把 ANN 图、聚类元数据或 Q/K OOD 校正放入 CXL pool；SAC 因此只覆盖 sparse attention + CXL 两个轴。
- 论文只评估 DeepSeek-V3.2，作者把其他模型、HBM/DRAM/CXL 混合分层列为后续工作。它是 arXiv 预印本，结果尚应按预印本证据看待。
- RDMA baseline 通过 ConnectX-7 loopback 与本地 DRAM 模拟，论文说明这是理想化的 best-case RDMA baseline；这一点影响跨架构性能比较的解释。见 Appendix A.2。

## 与本 wiki 的连接

- [DeepSeek-V3.2](deepseek-v3-2.md) 的 learned indexer 产生 SAC 所需 top-k；[ECHO](echo.md) 也服务 DSA，但用 host memory，而非 CXL pool。
- [Beluga](beluga.md) 研究 CXL KV pool 与通用 cache reuse；SAC 则关注 sparse decode 对 CXL 随机/细粒度读取的要求。
- [RetrievalAttention](retrievalattention.md) 把 sparse selection 与 ANNS 相连；将其 attention-aware index 与 SAC 的 CXL payload path 组合，是跨论文的待验证设计。
- 参见 [sparse attention、CXL 与 ANNS 综合页](../topics/sparse-attention-cxl-anns.md)。
