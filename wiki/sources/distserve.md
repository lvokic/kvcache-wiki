# DistServe: Disaggregating Prefill and Decoding for Goodput-optimized Large Language Model Serving

**作者：** Yinmin Zhong, Shengyu Liu, Junda Chen, Jianbo Hu, Yibo Zhu, Xuanzhe Liu, Xin Jin, Hao Zhang  
**发表：** OSDI 2024, pp. 193–210  
**本地 PDF：** [raw/papers/distserve-osdi24.pdf](../../raw/papers/distserve-osdi24.pdf)，USENIX 正式论文版  
**出处：** [USENIX OSDI 2024](https://www.usenix.org/conference/osdi24/presentation/zhong-yinmin) · [论文 PDF](https://www.usenix.org/system/files/osdi24-zhong-yinmin.pdf)

## 摘要

DistServe 把 prefill 与 decode 放到不同 GPU pool，以减少两阶段资源干扰，并分别优化 TTFT 与 TPOT。系统联合决定每个阶段的资源分配、并行策略和集群放置，以控制 P/D 分离引入的 KV 传输成本。详见原文 §3–4。

## 关键证据

论文报告，在其测试的模型、应用和延迟约束下，DistServe 相比当时的 serving 系统可承载最多 7.4 倍请求，或满足最高 12.6 倍严格的 SLO，同时超过 90% 请求满足延迟约束。该结果依赖实验中的集群、网络和 workload，不能当作跨论文的固定加速比。出处：[USENIX 摘要与论文](https://www.usenix.org/conference/osdi24/presentation/zhong-yinmin)。

## 系统视角

- 这是以请求为单位把生成出的 KV 从 prefill worker 交给 decode worker 的 disaggregated-pipeline 设计。
- 它主要解决阶段干扰、placement 和 SLO 优化；论文没有把跨会话的持久 KV store 作为中心抽象。和 [Mooncake](mooncake.md) 的 global cache 相比，这是 lifetime 与 reuse scope 的差别，而不是只换了传输协议。
- P/D 分离的收益需要和 KV bytes、互连带宽及 transfer latency 一起看；当传输代价过高，分离可能抵消计算隔离带来的收益。

