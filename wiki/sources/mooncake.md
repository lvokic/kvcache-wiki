# Mooncake: Trading More Storage for Less Computation — A KVCache-centric Architecture for Serving LLM Chatbot

**作者：** Ruoyu Qin, Zheming Li, Weiran He, Jialei Cui, Feng Ren, Mingxing Zhang, Yongwei Wu, Weimin Zheng, Xinran Xu  
**发表：** FAST 2025, pp. 155–170；Best Paper  
**本地 PDF：** [raw/papers/mooncake-fast25.pdf](../../raw/papers/mooncake-fast25.pdf)，USENIX 正式论文版  
**出处：** [USENIX FAST 2025](https://www.usenix.org/conference/fast25/presentation/qin) · [论文 PDF](https://www.usenix.org/system/files/fast25-qin.pdf)

## 摘要

Mooncake 是 Kimi 使用的 serving platform。它分离 prefill 和 decode pool，并把 CPU、DRAM、SSD 与 NIC/RDMA 资源组织成分布式 KV cache。全局 Conductor 选择 prefill/decode 实例、复用 prefix KV、流水式传递新 KV，并按冷热程度复制或换出 blocks。见原文 §3–4。

## 关键价值与证据

- 它把“多用存储、少做重复 prefill”作为系统设计中心：全局缓存命中率、跨节点传输、prefill 排队和 decode 中可容纳的 KV 一起进入调度目标。
- 论文在真实 trace 的实验中报告，在满足 SLO 的前提下，有效请求容量比其 baseline 高 59%–498%；实际部署数据也是作者报告的生产结果。两类证据的实验环境不同，引用时不应合并成一个普遍保证。出处：[USENIX 摘要和正式论文](https://www.usenix.org/conference/fast25/presentation/qin)。
- 这是 [DistServe](distserve.md) 的一个更完整组合：除 P/D 解耦和 KV handoff 外，还显式管理跨请求复用、存储层级和全局放置。

## 边界与问题

- 优势依赖高带宽互连、可用的 CPU/DRAM/SSD 容量、prefix reuse 机会和服务 SLO；这些条件需要与目标部署相匹配。
- 论文的系统级数值和部署规模来自作者自己的 trace、硬件和服务环境；其他系统论文的结果并未在同一配置下重跑。
- 和 [From Tensor Buffer survey](distributed-kv-hierarchy-survey.md) 的 taxonomy 对照时，Mooncake 的 store 子系统可看作 shared-store，但完整系统同时组合 P/D disaggregation 与 tiering，因此作者把它归为 hybrid-tier。

