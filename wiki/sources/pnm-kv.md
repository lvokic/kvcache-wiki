# Scalable Processing-Near-Memory for 1M-Token LLM Inference: CXL-Enabled KV-Cache Management Beyond GPU Limits

**作者：** Dowon Kim 等。**发表：** PACT 2025；arXiv:2511.00321。
**本地原件：** [pnm-kv-pact25.pdf](../../raw/papers/pnm-kv-pact25.pdf)
**来源：** [arXiv:2511.00321](https://arxiv.org/abs/2511.00321) · [DOI 10.1109/PACT65351.2025.00013](https://doi.org/10.1109/PACT65351.2025.00013)

## 摘要与设计

PNM-KV 将 KV management 和部分 attention computation 放到 CXL-attached processing-near-memory devices；并提出 GPU 与 PNM 协同的 hybrid execution，以扩展到 million-token inference。论文在自建 CXL-PNM 平台和系统规模分析中报告吞吐、能耗和成本收益。

## 对 proposal 的架构边界

- 该工作表明 CXL attention 的计算位置不只可在 GPU/CPU，也可用 endpoint PNM；因此要把本 proposal 限定为普通 memory-only CXL，说明不采用设备侧计算的理由。
- PNM-KV 依赖定制近存计算硬件，不能当成 commodity CXL memory 的直接性能基线；它的选择和执行粒度也与固定 selector 后的 group-level mixed routing 不同。
- 引用其数字时要分别说明原型和模型估算，避免把硬件预期结果写成通用 CXL 能力。

## 相关页面

- 参见 [COSMOS](cosmos-cxl-anns.md)、[SAC](sac.md)、[TRACE](trace-cxl.md) 和 [综合页](../topics/sparse-attention-cxl-anns.md)。
