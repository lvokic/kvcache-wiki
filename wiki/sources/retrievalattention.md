# RetrievalAttention: Accelerating Long-Context LLM Inference via Vector Retrieval

**作者：** Di Liu 等  
**发表：** NeurIPS 2025，主会  
**原始文件：** [raw/papers/retrievalatten.pdf](../../raw/papers/retrievalatten.pdf)  
**正式论文：** [NeurIPS 论文页与 PDF](https://proceedings.neurips.cc/paper_files/paper/2025/hash/4e36d4049fb0fea195a8267c8dcd0824-Abstract-Conference.html)

## 摘要

RetrievalAttention 是这批论文里 sparse attention 与 ANNS 直接结合的工作。它把固定上下文的多数 KV entries 和向量索引放在 CPU memory，decode 时按 query 检索相关 key，再取回相应 KV；少量静态 token 留在 GPU。作者观察到注意力中的 query 与 key 由不同投影产生，分布有 OOD 差异，普通 ANN 索引在此场景可能需要扫描大量 keys。

## 设计与主要结果

- 索引构建时，作者利用上下文中已有的 query/key 关系，将 query 到其 top-k key 的连接投影到 key 图上；这一投影步骤借鉴 [RoarGraph](roargraph.md)，避免在线保留 query 节点。见 §3.2、Fig. 4。
- 运行时按 query 搜索候选 KV；CPU 与 GPU 分别计算动态检索 token 和常驻静态 token 的部分 attention，再合并结果。固定开头 token 与最近窗口是其 GPU 静态集合的一种实现。见 §3.3。
- 在论文的长上下文测试中，索引检索约访问 1–3% 的 key vectors；128K、单 RTX 4090、8B 模型设置下，报告相较 exact-KNN 与传统 ANNS 分别达到 7.93× 和 2.80× decode speedup，准确率接近 full attention。见摘要、§4、Fig. 8–9。

## 边界与限制

- 论文针对 fixed context 可重复使用的场景进行离线索引构建；作者指出 prefill 没有优化，索引构建成本与更一般的动态上下文仍是限制。见 §F Limitations。
- 评估路径是 CPU memory 与 GPU 协同，并未评估 CXL。稀疏检索的比例、端到端速度和质量来自特定模型、任务、硬件与索引参数，不代表任意 serving workload。
- ANN 的 key recall 不是 attention 输出误差的充分刻画；实际系统仍需检查注意力权重/输出近似及下游任务质量。

## 与本 wiki 的连接

- [RoarGraph](roargraph.md) 是它明确引用的 query-aware 图投影来源；两者形成从通用 OOD ANNS 到 Q/K 检索索引的直接技术连接。
- [RetroInfer](retroinfer.md) 也把 KV 看作向量存储，但采用 attention-aware 聚类与估算，并联动 GPU/CPU buffer manager。
- 与 [CXL-ANNS](cxl-anns.md) 对照：后者优化 CXL 内存池上的通用图检索，不处理 Q/K OOD 或 KV attention 质量。
- [SAC](sac.md) 覆盖 sparse attention + CXL KV payload，但不含 ANNS；二者的组合是本 wiki 的待测路径。
- 参见 [sparse attention、CXL 与 ANNS 综合页](../topics/sparse-attention-cxl-anns.md)。
