# An Efficient Hybrid Sparse Attention with CPU-GPU Parallelism for Long-Context Inference

**作者：** Feiyu Yao, Zhixiong Niu, Xiaqing Li, Yongqiang Xiong, Juan Fang, Qian Wang。**版本：** arXiv:2605.07719 v1，2026 预印本；系统称为 Fluxion。
**本地原件：** [fluxion-arxiv26.pdf](../../raw/papers/fluxion-arxiv26.pdf)
**来源：** [arXiv:2605.07719](https://arxiv.org/abs/2605.07719)

## 摘要与设计

Fluxion 面向 CPU-resident KV 上的长上下文 decode。它组合 output-aware KV budget、head-property predictor、按 head 和粒度选择 sparse 配置，以及 CPU/GPU 优先级调度，以重叠选择、搬运与 attention 工作。论文报告在两种模型、三个 benchmark、40 项任务上，质量退化较小，并相对所测 fixed sparse hybrid baseline 提高 1.5–3.7 倍速度；速度结果来自作者定义的硬件和 budget 设置。

## 对 proposal novelty 的影响

- 这是本轮发现中最直接的“CPU/GPU 混合 sparse attention”先例。异构调度、按 head 配预算和让两侧同时做工作都不能声称首次。
- proposal 应比较固定 selector、固定每 head mask 下的执行 planner，并与 Fluxion 类 whole-group/task scheduling 做对照。可检验的窄点是：planner 是否根据每个 group 的实际驻留、冷 footprint、物理 span 与传输/合并成本，在 group 内切分 GPU packet 和 CPU partial work。
- Fluxion 还联合优化预算与 sparse 粒度；与其比较时应隔离“选集变化”与“同一选集的物理执行”两类收益。

## 相关页面

- 参见 [IceCache](icecache.md)、[SPIN](spin.md)、[ScoutAttention](scoutattention.md) 和 [Sparse Attention、CXL 与 ANNS 综合页](../topics/sparse-attention-cxl-anns.md)。
