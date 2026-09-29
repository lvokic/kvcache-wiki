# LLM KV Cache Wiki

本 wiki 聚焦 LLM inference/serving 中 KV cache 的内存管理和系统设计。当前材料包括两篇综述和 37 篇系统/算法论文，覆盖 GPU 本地分配、host offload、prefill/decode 解耦、跨请求缓存、网络传输、attention kernels，以及 sparse attention、ANNS 索引和 CXL memory tier。其中包括 owner 提供的 CXL-Vector SIGMOD ’27 匿名投稿稿；其状态在 [source note](sources/cxl-vector.md) 中单独注明。

## 当前工作假设

KV cache 不只是每个请求随生成增长的 tensor。现代系统会决定它放在哪里、保留多久、由谁管理、通过什么 substrate 访问；同一设计还会影响 TTFT、TPOT、吞吐和尾延迟。这个视角来自两篇互补综述：[三层优化 taxonomy](sources/survey-kv-cache-management.md) 和 [分布式内存层级 taxonomy](sources/distributed-kv-hierarchy-survey.md)。后者是 2026-06 的 arXiv v1 预印本。

## 从哪里读

- **先看全景：** [KV cache 管理与 serving 系统](topics/kv-cache-management.md)。
- **本地 GPU 分配：** [PagedAttention / vLLM](sources/pagedattention.md) 与 [vAttention](sources/vattention.md)。
- **offload 与远程传输：** [InfiniGen](sources/infinigen.md) 与 [CacheGen](sources/cachegen.md)。
- **serving 架构：** [DistServe](sources/distserve.md) 与 [Mooncake](sources/mooncake.md)。
- **kernel 数据路径：** [FlashInfer](sources/flashinfer.md)。
- **Sparse selection：** [Quest](sources/quest.md) → [DeepSeek-V3.2](sources/deepseek-v3-2.md)，对照 query-aware page bounds 和训练期 lightweight indexer。
- **近期 sparse serving systems：** [IceCache](sources/icecache.md)、[SPIN](sources/spin.md)、[HiSparse](sources/hisparse.md) 与 [Fluxion](sources/fluxion.md)，覆盖动态语义页、统一分层 cache、exact selection resolution 和 CPU/GPU hybrid attention。
- **Sparse attention + ANN：** [RetrievalAttention](sources/retrievalattention.md) → [RoarGraph](sources/roargraph.md)，关注 Q/K OOD 和 query-aware graph projection。
- **压缩键与质量边界：** [Louver](sources/louver.md)、[Verified vAttention](sources/vattention-verified.md)、[Self-Indexing KVCache](sources/self-indexing-kvcache.md) 与 [SALS](sources/sals.md)，区分阈值召回、近似误差保证和压缩表示索引。
- **索引 + 异构 KV 层级：** [RetroInfer](sources/retroinfer.md) 与 [ECHO](sources/echo.md)，对照 cluster index/buffer 管理和原生 sparse attention 的 host offload。
- **CXL + 图索引：** [CXL-ANNS](sources/cxl-anns.md)、[COSMOS](sources/cosmos-cxl-anns.md) 与用户提供的 [CXL-Vector](sources/cxl-vector.md)，对照 memory-only CXL、endpoint compute，以及 DRAM 导航/CXL 原向量重排。
- **Sparse attention + CXL：** [SAC](sources/sac.md) 使用 DSA top-k selector 并从 CXL 按需读取 KV；[Beluga](sources/beluga.md) 提供共享 CXL KV pool 的系统路径。两者都还没有集成 ANNS graph。
- **CXL 设备与数据路径：** [TRACE](sources/trace-cxl.md) 探索设备内压缩/精度分层；[PNM-KV](sources/pnm-kv.md) 把部分管理和计算放到近存设备。二者硬件假设需与 memory-only CXL 区分。

## 目前最有用的对比

1. **优化 locus vs. 系统语义：** token/model/system taxonomy 说明优化落在哪一层；locality/lifetime/ownership/substrate taxonomy 说明状态跨过哪些系统边界。两者可以叠加使用。
2. **请求级 handoff vs. 跨请求复用：** DistServe 关注把 prefill 产出的 KV 高效交给 decode；Mooncake 还增加跨请求 prefix reuse、全局放置和多层存储。
3. **分页管理 vs. 连续虚拟地址：** PagedAttention 让 attention kernel 遍历 block table；vAttention 利用 CUDA VMM 保留虚拟连续性，减少 kernel 对 paging 的显式支持。
4. **网络搬运 vs. 缓存控制面：** CacheGen 压缩和适配传输；Mooncake 管理全局缓存、放置与调度。二者在概念上可能组合，但本 wiki 尚无组合实验作为证据。
5. **选择器、索引、内存 tier 是不同设计轴：** RetrievalAttention 直接把 ANNS 用于 sparse attention；CXL-ANNS 优化的是 CXL 上通用 ANNS；ECHO 优化的是原生 sparse attention 的 KV offload。当前收录的论文未在同一系统里测量三者组合，见 [专题页](topics/sparse-attention-cxl-anns.md)。
6. **执行新颖性需在强先例下收窄：** HiSparse 已做 exact sparse cache resolution，Fluxion 已做 CPU/GPU hybrid sparse attention，IceCache 已做 GQA union 与 bulk backload，CXL-Vector 已做 memory-only CXL 候选精排。proposal 的待验证点应落在固定选集后按物理 footprint 的 group 内分流及其全成本收益，见 [专题评估](topics/sparse-attention-cxl-anns.md)。

## 资料范围

系统论文的 throughput、latency、容量和质量结果来自各自不同的模型、硬件、trace、网络和 SLO。当前不把这些数字放到统一排行榜中。2026 survey 还指出，远程 KV 访问、metadata path、完成开销、reuse distance、prefetchability、P99 归因和公开 trace 等测量常有缺口。来源页已把作者报告的结果与本 wiki 的跨论文推论分开。
