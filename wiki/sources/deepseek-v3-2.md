# DeepSeek-V3.2: Pushing the Frontier of Open Large Language Models

**作者：** DeepSeek-AI 及合作者  
**版本：** arXiv:2512.02556 v1，2025-12-02  
**原始文件：** [raw/deepseek_v3.2.pdf](../../raw/deepseek_v3.2.pdf)  
**正式记录：** [arXiv 论文与全文](https://arxiv.org/abs/2512.02556)

## 摘要

这份技术报告介绍 DeepSeek Sparse Attention（DSA），作为 DeepSeek-V3.2 相比 V3.1-Terminus 的架构改动。DSA 由 lightning indexer 和 fine-grained token selector 组成：indexer 从 query 与历史 token hidden states 计算分数，再对每个 query 选 top-k KV entries 执行主 attention。论文将主 attention 的复杂度从 $O(L^2)$ 降到 $O(Lk)$，其中 $k \ll L$；但 indexer 仍需对历史位置打分，复杂度仍为 $O(L^2)$，只是其 head 数较小并可用 FP8。见 §2.1、§2.3。

## 机制与结果

- DSA 的 indexer 有独立的轻量 query/key 投影与若干 indexer heads，以加权 ReLU 内积得出 token 分数。论文的 sparse training 阶段对每个 query 选择 2048 个 KV entries，并训练主模型适应该稀疏模式。见 §2.1.1。
- 作者报告长上下文推理成本有所下降，且与 V3.1-Terminus 相比，所评估的短/长上下文能力没有明显退化。报告涵盖继续预训练、后训练与模型能力，不能把结果单独归因于 sparse attention。见 §2.2–2.3。

## 边界与限制

- 这是训练期与模型架构协同设计的 sparse attention，不是可直接套在任意预训练模型上的通用 ANN 数据库；lightning indexer 的全历史打分仍是二次复杂度。
- 报告没有评估 CXL 内存池、外置 ANNS 图索引或远程 KV 查询路径。DSA 的 lightweight learned indexer 与 ANNS 的存储图索引是两种不同设计。
- 报告里的 2048 预算属于其训练配置；不能据此假定任意模型/任务都适用同一固定预算。

## 与本 wiki 的连接

- [ECHO](echo.md) 直接面向原生 sparse attention 模型的 serving 与 offload，使用 DeepSeek-V3.2 作为重要 workload。
- [SAC](sac.md) 将 DSA 输出的 top-k selection 用于 CXL KV fetch，但没有替换 DSA indexer 为 ANNS。
- 与 [Quest](quest.md) 对照 query-aware 的动态选择方法；与 [RetrievalAttention](retrievalattention.md) 对照 learned indexer 和 ANNS 索引。
- 这份报告是 [sparse attention、CXL 与 ANNS 综合页](../topics/sparse-attention-cxl-anns.md) 的模型侧锚点。
