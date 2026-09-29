# SWARM: Co-Activation Aware KVCache Offloading Across Multiple SSDs

**作者：** Tuowei Wang, Liyun Chu, Ruwen Fan, Ju Ren。**版本：** arXiv:2603.17803 v1，2026 预印本。
**本地原件：** [swarm-arxiv26.pdf](../../raw/papers/swarm-arxiv26.pdf)
**来源：** [arXiv:2603.17803](https://arxiv.org/abs/2603.17803)

## 摘要与设计

SWARM 观察真实负载中 KV entries 会稳定地共同激活，并把这种 co-activation 用于多 SSD offload：离线聚类后跨设备布局并选择性复制，运行时负载均衡读取，并动态调整 cluster/cache。目标是把单设备带宽瓶颈转成可并行的 I/O。

## 与本 proposal 的联系

- co-access 关系、聚类布局和复制是有先例的，因此“按共访问关系放置 KV”不是空白。
- 它针对 SSD 容量和多设备并行 I/O，不是 CXL random-access pool，也不在固定 attention mask 下比较同 group 的 CPU/GPU partial route。其共激活模式能否由单 query 的 per-head attention selection 推断，仍需实测。
- 作为布局参考，应把 placement 收益和本 proposal 的在线物理规划收益分开测量。

## 相关页面

- 参见 [IceCache](icecache.md)、[Strata](strata.md) 与 [Sparse Attention、CXL 与 ANNS 综合页](../topics/sparse-attention-cxl-anns.md)。
