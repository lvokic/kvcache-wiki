# Exploring CXL-based KV Cache Storage for LLM Serving

**作者：** Yupeng Tang 等。**发表：** NeurIPS 2024 Machine Learning for Systems workshop。
**本地原件：** [exploring-cxl-kv-storage-neurips24.pdf](../../raw/papers/exploring-cxl-kv-storage-neurips24.pdf)
**论文 PDF：** [ML for Systems workshop copy](https://mlforsystems.org/assets/papers/neurips2024/paper17.pdf)

## 摘要与设计

本文早期评估 CXL memory 用作 LLM serving 的 KV/prefix cache 存储。它比较 CXL–CPU 与 CPU–GPU 传输表现，并在相同 SLO 下对比保存 prefix KV 与重新计算；作者报告 CXL cache 能提高可服务 batch，并给出生产部署的 ROI 建模。文中的最高容量和利用率收益属于其 workload 与模型假设，不是稀疏 decode 的实测结论。

## 与本 proposal 的边界

- 这是 CXL KV capacity 与复用价值的基础系统对照，应帮助解释 CXL 的定位和经济性。
- 关注的是跨请求 prefix reuse 和避免 prefill 重算；不涉及 query-dependent sparse decode、ANN selection、CPU partial attention 或 group-internal physical planner。
- 不能用其带宽/ROI 结论代替按需读取 KV 的 CXL latency、tail、事务放大和 decode throughput 实验。

## 相关页面

- 参见 [Beluga](beluga.md)、[SAC](sac.md)、[TRACE](trace-cxl.md) 和 [综合页](../topics/sparse-attention-cxl-anns.md)。
