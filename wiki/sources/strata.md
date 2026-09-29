# Strata: Hierarchical Context Caching for Long Context Language Model Serving

**作者：** Zhiqiang Xie 等。**发表：** OSDI 2026。
**本地原件：** [strata-osdi26.pdf](../../raw/papers/strata-osdi26.pdf)
**正式来源：** [USENIX OSDI 2026 paper page](https://www.usenix.org/conference/osdi26/presentation/xie-zhiqiang)

## 摘要与设计

Strata 研究长上下文 serving 的分层 context cache。其设计分离 GPU 与 host 的数据布局，并使用 GPU-assisted I/O、缓存感知调度和 delayed-hit 管理搬运及 cache miss；论文描述了 SGLang 集成和端到端生产负载评估。

## 对本 proposal 的边界

- 说明 host/GPU layout decoupling、大范围数据搬运重叠和 cache-aware scheduling 均有 OSDI 系统先例。
- Strata 的中心是 context/prefix cache 与较大范围载入，不是 CXL memory-only 的逐层 decode sparse selection，也不比较按固定逐 head mask 在 CPU/GPU 间做 partial attention。
- 本 proposal 若使用 layout conversion 或 prefetch，应以 Strata 为系统基线，区分全局 cache I/O 与 decode 每步少量非连续 KV 的成本。

## 相关页面

- 参见 [DirectKV](directkv.md)、[SPIN](spin.md) 与 [KV cache 管理总览](../topics/kv-cache-management.md)。
