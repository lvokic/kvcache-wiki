# IceCache: Memory-efficient KV-cache Management for Long-Sequence LLMs

**作者：** Yuzhen Mao, Qitong Wang, Martin Ester, Ke Li。**发表：** ICLR 2026。
**本地原件：** [icecache-iclr26.pdf](../../raw/papers/icecache-iclr26.pdf)
**正式来源：** [ICLR 2026 proceedings](https://proceedings.iclr.cc/paper_files/paper/2026/hash/94de1ef32f1b564b885720ab89fd95af-Abstract-Conference.html) · [arXiv:2604.10539](https://arxiv.org/abs/2604.10539)

## 摘要与设计

IceCache 将语义相近的 KV tokens 聚成连续区域，并以可动态更新的 DCI 树映射 key IDs 到存储页。运行时检索出 tokens 后，系统通过 GQA-aware union 合并读取，并用 backload buffer 批量搬回 GPU。论文旨在让 CPU-hosted KV 的选择和搬运获得更好的 locality。

论文在 LongBench 上报告：256-token budget 下达到 full-KV 模型约 99% 的原始准确度；与其对比的 offloading 方法相比，使用约 25% token budget 时延迟和准确度有竞争力。结果受模型、任务、budget 定义及实验平台限制，不能直接等同于 attention 输出误差。

## 对当前研究的意义

- 语义分组、动态目录、GQA union 和批量 backload 都是已有设计；这些应成为物理规划的强 baseline。
- IceCache 的主线是选择与搬运的数据组织，不是按已固定的逐 head 选集和真实驻留 footprint 将一个 KV group 拆到 CPU partial 与 GPU packet 两条计算路径。
- 见论文方法和评估部分；需在相同选集、精度、容量约束下比较，避免把选择预算差异误算成系统收益。

## 相关页面

- [SPIN](spin.md)、[HiSparse](hisparse.md) 和 [Fluxion](fluxion.md) 分别覆盖分层 cache 管理、精确 cache resolve 和 CPU/GPU 混合 sparse attention。
- 参见 [Sparse Attention、CXL 与 ANNS 综合页](../topics/sparse-attention-cxl-anns.md)。
