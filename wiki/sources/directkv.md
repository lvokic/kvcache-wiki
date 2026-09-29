# No Buffer, No Bottleneck: Efficient Zero-Copy KV Cache Offloading for Long-Context LLMs

**作者：** Shutian Luo, Haiying Shen。**发表：** OSDI 2026；系统名 DirectKV。
**本地原件：** [directkv-osdi26.pdf](../../raw/papers/directkv-osdi26.pdf)
**正式来源：** [USENIX OSDI 2026 paper page](https://www.usenix.org/conference/osdi26/presentation/luo)

## 摘要与设计

DirectKV 面向 GH200/GB200 等 NVLink-C2C heterogeneous CPU–GPU 平台，允许 GPU kernel 直接访问 CPU KV，省去 staging buffer。论文通过重新设计矩阵访问模式、warp-level pipeline 和 fused kernels，将数据传输与计算重叠；GH200 评估报告最高约 43% GPU memory reduction、1.2 倍端到端性能提升。

## 适用边界

- 直接 CPU-memory access、warp pipeline 与融合避免重复读取，是 host-tier kernel 设计的重要 prior。
- 结果依赖 NVLink-C2C 互连与其带宽/延迟特性，不能外推为 PCIe+CXL 结果；论文没有 attention ANNS 或 CXL pool。
- 它应作为强 host-memory whole-path baseline，proposal 需分清“远端原地读”与“显式 packet 搬运/CPU partial”在真实 PCIe+CXL 平台上的负担。

## 相关页面

- 参见 [Strata](strata.md)、[ScoutAttention](scoutattention.md) 和 [综合页](../topics/sparse-attention-cxl-anns.md)。
