# MiniMax Sparse Attention

**版本：** arXiv:2606.13392 v2，2026-06-12 预印本。
**本地原件：** [minimax-sparse-attention-arxiv26.pdf](../../raw/papers/minimax-sparse-attention-arxiv26.pdf)
**来源：** [arXiv:2606.13392](https://arxiv.org/abs/2606.13392)

## 摘要与设计

MiniMax Sparse Attention（MSA）为 GQA 每个 KV group 配置独立 block-level index branch，按 group 选取 main attention 所需 key blocks，并以训练期 KL alignment 监督 selector。执行路径使用 exp-free top-k 和 KV-outer sparse kernel。论文报告 109B 模型在 1M context 下相对 GQA full attention 减少 28.4 倍 attention compute；该结果建立在模型训练和模型—kernel 联合设计之上。

## 与本研究的关系

- MSA 是 group-specific selector 的代表，说明同一 KV group 的选择可能依赖该组训练出的索引语义。它适合作为选择器与 mask 语义的背景，但它不讨论 CXL 放置或 CPU/GPU 物理规划。
- 本 proposal 的 planner 应严格保留 selector 产生的逐 head/group mask；不能因物理 union 而把 MSA 的选择语义改成所有 heads 都访问 union。
- 这是模型方法与 kernel 共设计工作，不能将其质量、context 和加速结果与固定模型的系统执行优化直接对比。

## 相关页面

- 参见 [DeepSeek-V3.2](deepseek-v3-2.md)、[CompactAttention](compactattention.md) 和 [Sparse Attention、CXL 与 ANNS 综合页](../topics/sparse-attention-cxl-anns.md)。
