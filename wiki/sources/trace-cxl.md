# TRACE: Unlocking Effective CXL Bandwidth via Lossless Compression and Precision Scaling

**作者：** Rui Xie 等。**版本：** arXiv:2509.03377 v3，2026-01-30；该版本 PDF 标题为 TRACE，早期版本曾以 CXL-NDP 标题流传。
**本地原件：** [trace-cxl-arxiv26.pdf](../../raw/papers/trace-cxl-arxiv26.pdf)
**来源：** [arXiv:2509.03377](https://arxiv.org/abs/2509.03377)

## 摘要与设计

TRACE 保留标准 CXL.mem 接口，调整设备内部 tensor 表示为 channel-major、disaggregated bit-plane layout；对 KV 做专用 transform 后进行通用无损压缩，并可按精度需求只读取部分 bit planes。v3 报告 BF16 KV footprint 无损降低 46.9%；trace-driven system modeling 在 CXL spill 情况下对特定模型/128K 设置报告最高 4.24 倍 throughput。论文也报告 DRAMSim3 能耗模型与 7 nm RTL synthesis 结果。

## 与本 proposal 的关系

- TRACE 直接挑战“CXL 读回的 payload bytes/precision 必须照搬常规布局”的假设；若研究 sparse physical planner，需注明其 bytes 与 precision 视图如何与设备内部压缩交互。
- 这是 near-data storage controller 与布局方案，并非 attention-aware ANN，也没有在真实 commodity CXL device 上验证完整 serving kernel。系统收益是 trace/model-based，不可直接视为可部署测量。
- 新版本标题为 TRACE；避免把该 PDF 错记为不同的 CXL-NDP 论文或混用版本间数字。

## 相关页面

- 参见 [CXL storage exploration](exploring-cxl-kv-storage.md)、[SAC](sac.md) 和 [综合页](../topics/sparse-attention-cxl-anns.md)。
