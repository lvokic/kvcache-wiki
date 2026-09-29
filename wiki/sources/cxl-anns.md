# CXL-ANNS: Software-Hardware Collaborative Memory Disaggregation and Computation for Billion-Scale Approximate Nearest Neighbor Search

**作者：** Junhyeok Jang, Hanjin Choi, Hanyeoreum Bae, Seungjun Lee, Miryeong Kwon, Myoungsoo Jung  
**发表：** USENIX ATC 2023，585–600  
**本地原件：** [raw/papers/cxl-anns-atc23.pdf](../../raw/papers/cxl-anns-atc23.pdf)  
**正式论文：** [USENIX 论文页与 PDF](https://www.usenix.org/conference/atc23/presentation/jang)

## 摘要

CXL-ANNS 研究如何让 billion-point graph ANNS 使用可扩展的 CXL memory pool，同时缓解 CXL 远端访问造成的随机图遍历延迟。它是本批里 CXL 与 ANN 索引结合的直接系统案例，但没有 LLM、KV cache 或 attention。

## 设计与结果

- 将大部分 graph/vector data 放入 CXL pool；按图节点到 entry node 的 hop 关系，把预期高频访问的邻居信息缓存在本地 DRAM。对可能访问的下一批邻居做预取，并把距离计算与图遍历/候选更新分配给 CXL endpoint 与 host，降低数据来回搬运和硬件空闲。见 §3–5、Fig. 13–19。
- 作者先观察到朴素 CXL 基线相对无限 DRAM oracle 最多约 3.9× 慢；这说明“容量可扩展”本身不消除 ANN 图访问的远端延迟。完整方案的关键是关系感知本地缓存、预测预取、endpoint 协同计算及细粒度调度。
- 评估结合 Linux/16nm FPGA 原型与 gem5 全系统模拟，在六个 billion-point 数据集上展开；论文报告相较所测 state-of-the-art ANNS 平台最高 111.1× QPS、93.3% 更低 query latency。该极值属于指定 baseline/configuration 的比较，不应外推为普遍增益。见 §1、§6。

## 边界与限制

- CXL-ANNS 不处理 attention query/key 的 OOD 分布，也不检验 ANN recall 是否足以保持 attention 输出和语言任务质量。
- 本文是通用 vector graph search 的 CXL 设计；它的本地 DRAM caching 与预取机制可以启发 KV 检索系统，但需要在 attention 的 per-layer/per-head workload 下重新评估。

## 与本 wiki 的连接

- [RetrievalAttention](retrievalattention.md) 提供注意力特有的 query–key OOD 索引；CXL-ANNS 提供图索引跨本地 DRAM/CXL 的放置、预取和协同搜索机制。两者组合是推论，尚非本论文评测结果。
- [Beluga](beluga.md) 与 [SAC](sac.md) 分别说明 CXL 如何支持共享 KV pool 和 sparse KV 按需读取；它们均未将 ANNS index 放入 CXL。
- [From Tensor Buffer survey](distributed-kv-hierarchy-survey.md) 将 memory pool 作为 KV substrate 分类；[ECHO](echo.md) 则展示原生稀疏注意力在 host memory 上的 offload serving。
- 参见 [sparse attention、CXL 与 ANNS 综合页](../topics/sparse-attention-cxl-anns.md)。
