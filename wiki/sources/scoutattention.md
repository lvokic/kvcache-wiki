# ScoutAttention: Efficient KV Cache Offloading via Layer-Ahead CPU Pre-computation for LLM Inference

**作者：** Qiuyang Zhang 等。**发表：** DAC 2026。
**本地原件：** [scoutattention-dac26.pdf](../../raw/papers/scoutattention-dac26.pdf)
**来源：** [arXiv:2603.27138](https://arxiv.org/abs/2603.27138)

## 摘要与设计

ScoutAttention 将部分 KV attention 放在 CPU 上，并利用 layer-ahead pre-computation：以预测的下一层 query 提前执行 CPU 工作，再通过异步 periodic recall 与 GPU 路径协同。论文结论报告相对 full attention 约 2.1% accuracy drop、最高 5.1 倍 speedup。预测 query 和实际 query 的差异是该路径的重要语义与质量约束，不能把预计算直接视作精确的 partial attention。

## 对 proposal 的影响

- CPU 与 GPU 协同 attention、跨层预先计算和异步 recall 都已有先例；不应声称首次使用 CPU 计算 sparse attention。
- 可比较的边界是预测驱动的 layer-ahead 路径与 selection 已固定后按实际地址/驻留进行的精确分区。应分别报告预测质量误差、partial-attention 归并成本和冷数据搬运。
- 该论文聚焦 host KV offload，不是 CXL memory-only 池或 ANNS index。

## 相关页面

- 参见 [Fluxion](fluxion.md)、[HiSparse](hisparse.md) 和 [综合页](../topics/sparse-attention-cxl-anns.md)。
