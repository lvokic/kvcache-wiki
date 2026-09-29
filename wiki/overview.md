# LLM KV Cache Wiki

本 wiki 聚焦 LLM inference/serving 中 KV cache 的内存管理和系统设计。当前材料包括两篇综述和十六篇系统/算法论文，覆盖 GPU 本地分配、host offload、prefill/decode 解耦、跨请求缓存、网络传输、attention kernels，以及本阶段重点：sparse attention、ANNS 索引和 CXL memory tier。

## 当前工作假设

KV cache 不只是每个请求随生成增长的 tensor。现代系统会决定它放在哪里、保留多久、由谁管理、通过什么 substrate 访问；同一设计还会影响 TTFT、TPOT、吞吐和尾延迟。这个视角来自两篇互补综述：[三层优化 taxonomy](sources/survey-kv-cache-management.md) 和 [分布式内存层级 taxonomy](sources/distributed-kv-hierarchy-survey.md)。后者是 2026-06 的 arXiv v1 预印本。

## 从哪里读

- **先看全景：** [KV cache 管理与 serving 系统](topics/kv-cache-management.md)。
- **本地 GPU 分配：** [PagedAttention / vLLM](sources/pagedattention.md) 与 [vAttention](sources/vattention.md)。
- **offload 与远程传输：** [InfiniGen](sources/infinigen.md) 与 [CacheGen](sources/cachegen.md)。
- **serving 架构：** [DistServe](sources/distserve.md) 与 [Mooncake](sources/mooncake.md)。
- **kernel 数据路径：** [FlashInfer](sources/flashinfer.md)。
- **Sparse selection：** [Quest](sources/quest.md) → [DeepSeek-V3.2](sources/deepseek-v3-2.md)，对照 query-aware page bounds 和训练期 lightweight indexer。
- **Sparse attention + ANN：** [RetrievalAttention](sources/retrievalattention.md) → [RoarGraph](sources/roargraph.md)，关注 Q/K OOD 和 query-aware graph projection。
- **索引 + 异构 KV 层级：** [RetroInfer](sources/retroinfer.md) 与 [ECHO](sources/echo.md)，对照 cluster index/buffer 管理和原生 sparse attention 的 host offload。
- **CXL + 图索引：** [CXL-ANNS](sources/cxl-anns.md)，观察 CXL 的随机访问成本、局部缓存、预取和 endpoint 协同计算。
- **Sparse attention + CXL：** [SAC](sources/sac.md) 使用 DSA top-k selector 并从 CXL 按需读取 KV；[Beluga](sources/beluga.md) 提供共享 CXL KV pool 的系统路径。两者都还没有集成 ANNS graph。

## 目前最有用的对比

1. **优化 locus vs. 系统语义：** token/model/system taxonomy 说明优化落在哪一层；locality/lifetime/ownership/substrate taxonomy 说明状态跨过哪些系统边界。两者可以叠加使用。
2. **请求级 handoff vs. 跨请求复用：** DistServe 关注把 prefill 产出的 KV 高效交给 decode；Mooncake 还增加跨请求 prefix reuse、全局放置和多层存储。
3. **分页管理 vs. 连续虚拟地址：** PagedAttention 让 attention kernel 遍历 block table；vAttention 利用 CUDA VMM 保留虚拟连续性，减少 kernel 对 paging 的显式支持。
4. **网络搬运 vs. 缓存控制面：** CacheGen 压缩和适配传输；Mooncake 管理全局缓存、放置与调度。二者在概念上可能组合，但本 wiki 尚无组合实验作为证据。
5. **选择器、索引、内存 tier 是不同设计轴：** RetrievalAttention 直接把 ANNS 用于 sparse attention；CXL-ANNS 优化的是 CXL 上通用 ANNS；ECHO 优化的是原生 sparse attention 的 KV offload。当前收录的论文未在同一系统里测量三者组合，见 [专题页](topics/sparse-attention-cxl-anns.md)。

## 资料范围

系统论文的 throughput、latency、容量和质量结果来自各自不同的模型、硬件、trace、网络和 SLO。当前不把这些数字放到统一排行榜中。2026 survey 还指出，远程 KV 访问、metadata path、完成开销、reuse distance、prefetchability、P99 归因和公开 trace 等测量常有缺口。
