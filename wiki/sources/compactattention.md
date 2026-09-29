# CompactAttention: Accelerating Chunked Prefill with Block-Union KV Selection

**作者：** Jiwon Song, Dongwon Jo, Beomseok Kang, Jae-Joon Kim。**版本：** arXiv:2605.16839 v1，2026 预印本。
**本地原件：** [compactattention-arxiv26.pdf](../../raw/papers/compactattention-arxiv26.pdf)
**来源：** [arXiv:2605.16839](https://arxiv.org/abs/2605.16839)

## 摘要与设计

CompactAttention 面向 chunked prefill：将 2D block-sparse masks 当作 KV 选择信号，经 query-block union 和 GQA 内 union 转成 paged execution 的 KV block tables，从而原地访问所选块而不显式 compact KV。论文称生成的 block table 保留输入 masks 选中的 KV blocks，并报告 LLaMA-3.1-8B-Instruct、RULER 128K 上最高 2.72 倍 attention 加速。对 GQA 比例较大的模型，论文把 group 再分成四头 subgroups，以减轻 full-group union 的稀疏度损失；增加 metadata 与表构造开销。

## 边界与相关性

- 证明 mask-to-page-table、GQA/subgroup union 和零拷贝 paged access 已有 chunked-prefill 先例。
- 它是多 query 的 prefill 方法，不是 decode 时将冷 KV group 的不同物理片段分到 CPU partial 与 GPU 路径；union 保留 blocks 的覆盖性质也不代表原逐 head mask 语义完全等价。
- 其方法可作为 packet/table 构造与 GQA 复用的邻近基线，而不应和 decode 的端到端结果直接比较。

## 相关页面

- 参见 [FlashInfer](flashinfer.md)、[Fluxion](fluxion.md) 与 [Sparse Attention、CXL 与 ANNS 综合页](../topics/sparse-attention-cxl-anns.md)。
