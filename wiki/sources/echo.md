# ECHO: Efficient KV Cache Offloading with Lossless Prefetching for Serving Native Sparse Attention LLMs

**作者：** Guangda Liu, Wenhao Chen, Chengwei Li, Zhenyu Ning, Jing Lin, Yiwu Yao, Quan Chen, Shixuan Sun, Jieru Zhao, Minyi Guo  
**发表：** OSDI 2026，17–37  
**本地原件：** [raw/papers/echo-osdi26.pdf](../../raw/papers/echo-osdi26.pdf)  
**正式论文：** [USENIX 论文页与 PDF](https://www.usenix.org/conference/osdi26/presentation/liu-guangda)

## 摘要

ECHO 研究一个容易被“稀疏 attention 已经少读 KV”掩盖的问题：native sparse attention 仍需保存长上下文 KV 和 indexer key，HBM 容量会限制并发。ECHO 把 KV 部分放入 host memory，用 GPU-resident cache 服务选中的 tokens，并使动态 eviction/recall 与 CUDA Graph 兼容。

## 设计与结果

- **Graph-friendly cache manager：** 用 GPU 上可执行的 tensor metadata 管理 token/block 的 host 与 GPU 映射，使 decode 的分配、淘汰和 recall 能放进图执行路径。GPU kernel 可经 Unified Virtual Memory 访问 host pool。见 §4。
- **Lossless prefetch：** 利用 indexer score 的数值规律，在 decode 计算 top-k indexer scores 时提前召回必需 KV；prefill 则利用 query 顺序做跨 query 预取。系统把传输和 indexer 计算 pipeline overlap。见 §5。
- 在 4-bit DeepSeek-V3.2-Exp、8×H20、约 1TB host KV pool 等设置下，论文报告长上下文 workload 中最高 2.1× generation throughput（相较 SGLang），轻载 latency 接近；host pool 设置约 1.8M tokens。见 §6–7、Fig. 10–12。

## 边界与限制

- ECHO 的 host offload 依赖 CPU host memory 与 GPU–host interconnect；论文没有评估 CXL memory pool，也没有引入 ANNS 图索引。它使用 DSA indexer 的 token scores，并针对已选 token 做 cache recall/prefetch。
- 实验重点是 DeepSeek sparse attention 的 serving；模型权重使用量化配置，结果依赖硬件、SGLang/DeepGEMM 实现、负载与 host pool 大小。
- 论文指出 recall 与 cache management 本身仍会增加 ITL；所报告吞吐提升伴随 workload/load-dependent latency trade-off。见 §7。

## 与本 wiki 的连接

- [DeepSeek-V3.2](deepseek-v3-2.md) 提供 ECHO 服务的原生 sparse attention/indexer；ECHO 将 focus 从 token selection 推到容量、cache management 和 host 数据路径。
- [SAC](sac.md) 将 host offload 改为 CXL pool 上按需读取 sparse KV；两者可以对比 host-memory prefetch 与 CXL fine-grained access 的 trade-off。
- 与 [RetroInfer](retroinfer.md) 比较：两者都把 KV 放到 CPU 侧并管理 GPU 缓存；RetroInfer 用 wave index 估算与聚类，ECHO 使用 DSA top-k 分数和无损预取。
- 参见 [sparse attention、CXL 与 ANNS 综合页](../topics/sparse-attention-cxl-anns.md)。
