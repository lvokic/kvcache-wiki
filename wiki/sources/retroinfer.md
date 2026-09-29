# RetroInfer: A Vector Storage Engine for Scalable Long-Context LLM Inference

**作者：** Yaoqi Chen 等  
**发表：** PVLDB 19(5)，2026；DOI 10.14778/3796195.3796212  
**本地版本：** [raw/papers/retroinfer.pdf](../../raw/papers/retroinfer.pdf)，arXiv:2505.02922 v3  
**正式记录：** [PVLDB DOI](https://doi.org/10.14778/3796195.3796212) · [arXiv 论文](https://arxiv.org/abs/2505.02922)

## 摘要

RetroInfer 将 KV cache 作为 vector storage 来管理，联合设计 Attention-aWare VEctor index（wave index）和 GPU/CPU buffer manager（wave buffer）。它想解决的不只是“选中哪些 token”，还包括不同 layer/head/step 的 sparsity 变化、索引候选的准确率与搬运成本之间的取舍，以及如何把 host 侧 KV 有效送进 GPU。

## 设计

- **Wave index：** 用 spherical k-means 将 key vectors 分成 cluster；GPU 上保留 centroids 与元数据，CPU memory 中的 KV 按连续 blocks 存储。方法包含 tripartite attention approximation、accuracy-bound attention estimation 和 segmented clustering。对重要 cluster 做精确 attention，对其余区域用受界估计覆盖注意力贡献。见 §4.1–4.2。
- **Wave buffer：** 在 GPU 管理 KV block cache 和 attention execution buffer；CPU 端管理控制元数据与异步 replacement。计算与 GPU–CPU 数据搬运重叠；索引构建可与 prefill/offload 并行。见 §4.3–4.6。
- 论文在 A100 + 大容量 CPU DRAM、PCIe 4.0 设置中报告：120K 上相较 full attention 的 decode throughput 最多提高 4.4×，1M 上相较 sparse baselines 最多提高 12.2×，并报告达到 full-attention-level accuracy。数值来自论文自己的模型、benchmark、设备和配置，不是跨论文排名。

## 边界与限制

- 它采用 cluster-based attention-aware index，而非 RoarGraph 式图索引；ANN 构造、准确率估计与数据搬运共同影响结果。
- 论文实测的是 GPU–CPU/PCIe 路径，未评估 CXL memory pool。把 wave index/buffer 移植到 CXL 是待测系统假设，不是论文结果。
- 结果受预设模型、数据集、segment/cluster 参数与主机互连影响；CXL 随机访问、远端控制路径和共享池争用未覆盖。

## 与本 wiki 的连接

- [RetrievalAttention](retrievalattention.md) 侧重 OOD-aware graph retrieval；RetroInfer 侧重 attention-aware cluster index 和异构 buffer 管理。
- [ECHO](echo.md) 侧重 native sparse attention 下的 serving cache/offload 与精确预取。
- 参见 [sparse attention、CXL 与 ANNS 综合页](../topics/sparse-attention-cxl-anns.md)。
