# Sparse Attention、CXL 与 ANNS：索引、存储层级和可组合路径

## 范围与结论

本页关注三个组件如何相接：**sparse attention** 决定一次 attention 需要哪些 token；**ANNS index** 可用向量检索近似找到相关 keys；**CXL** 可扩展索引与 KV 的内存容量，但远端随机访问仍有代价。

当前文献集已覆盖三个两两组合：[RetrievalAttention](../sources/retrievalattention.md) 将 sparse attention 和 attention-aware ANNS 结合；[SAC](../sources/sac.md) 将 sparse attention 的 top-k KV 读取放到 CXL；[CXL-ANNS](../sources/cxl-anns.md) 将 ANN graph 放到 CXL 并优化遍历。**目前没有一篇在同一系统和 workload 中评估 sparse attention + CXL + ANNS 的完整组合。**

| 论文 | Sparse attention | CXL | ANNS/vector index | 系统边界 |
|---|---|---|---|---|
| [RetrievalAttention](../sources/retrievalattention.md) | 是 | 否，CPU memory | 是，attention-aware key graph | sparse retrieval + ANN；固定上下文离线建索引 |
| [CXL-ANNS](../sources/cxl-anns.md) | 否 | 是 | 是，通用 graph ANNS | 不处理 KV 或 query/key OOD |
| [SAC](../sources/sac.md) | 是，DeepSeek DSA | 是 | 否，使用模型的 top-k indexer | CXL 负责 KV payload 按需读取，不含 ANN 图搜索 |
| [Beluga](../sources/beluga.md) | 未作 end-to-end sparse attention | 是 | 否；论文将 vector/graph DB 列为潜在方向 | 多主机共享 CXL KV pool，并测 sparse-token 传输 microbenchmark |
| [ECHO](../sources/echo.md) | 是，native sparse | 否，host memory | 否，使用 DSA indexer | offload/cache manager/prefetch，不是 CXL pool |
| [RetroInfer](../sources/retroinfer.md) | 是，dynamic sparsity | 否，CPU memory | 是，cluster-based wave index | index 与 GPU/CPU buffer manager 联合设计 |
| [CXL-Vector](../sources/cxl-vector.md) | 否 | 是，memory-only CXL | 是，HNSW/NSG | 本仓库用户提供的匿名 SIGMOD ’27 稿件；DRAM 压缩导航、CXL 原向量候选重排，尚非 attention 工作 |
| [COSMOS](../sources/cosmos-cxl-anns.md) | 否 | 是，CXL endpoint | 是，通用 graph ANNS | CXL device 内有通用核心，和 memory-only 路线硬件假设不同 |
| [IceCache](../sources/icecache.md) | 是，token selection | 否，CPU/DRAM | 否 | 动态语义页、GQA union、批量 backload |
| [SPIN](../sources/spin.md) | 是，多种 sparse selector | 否，host memory | 否 | 统一 page substrate、动态 HBM budget 和分层 metadata |
| [Fluxion](../sources/fluxion.md) | 是，hybrid sparse attention | 否，CPU/GPU host path | 否 | output-aware budget、granularity selection、CPU/GPU 调度 |
| [HiSparse](../sources/hisparse.md) | 是，exact selection resolve | 否，host memory | 否 | bounded GPU cache 与 exact layerwise fetch/prefetch |
| [ScoutAttention](../sources/scoutattention.md) | 是，CPU/GPU co-attention | 否，host memory | 否 | layer-ahead query 预测和异步 recall |
| [CompactAttention](../sources/compactattention.md) | 是，chunked prefill | 否 | 否 | mask 转 GQA-aware paged block tables；属于 prefill 场景 |
| [Exploring CXL KV Storage](../sources/exploring-cxl-kv-storage.md) | 否，prefix/KV reuse | 是 | 否 | CXL KV capacity 与复用经济性，不含 sparse decode |
| [PNM-KV](../sources/pnm-kv.md) | 有 token/page selection | 是，CXL-attached PNM | 否 | 选择与 attention 部分移到定制近存加速器 |
| [TRACE](../sources/trace-cxl.md) | 否 | 是，CXL.mem | 否 | 设备内部 bit-plane layout、压缩和精度相关读取 |

这张矩阵区分两种常被统称为“index”的组件：DSA 的 learned score indexer 产生 token 排序；ANN/vector index 负责从大量向量中找到候选。SAC 已展示前者与 CXL KV pool 配合，RetrievalAttention 展示后者用于 attention token selection；把后者放进 CXL 则要额外解决图遍历的远端指针依赖。

## 三层设计空间

| 层次 | 论文证据 | 系统选择 |
|---|---|---|
| 选哪些 KV | [Quest](../sources/quest.md) 用 query 给 KV pages 排序；[DeepSeek-V3.2](../sources/deepseek-v3-2.md) 用 trained lightning indexer 对 tokens 打分；[RetrievalAttention](../sources/retrievalattention.md) 用 ANNS 搜索 keys | page bound、trained indexer，或 attention-aware vector index；这些方法不是同一种索引 |
| 如何遍历索引 | [RoarGraph](../sources/roargraph.md) 用 query distribution 改善 OOD graph traversal；RetrievalAttention 明确借用其 bipartite projection；[RetroInfer](../sources/retroinfer.md) 使用 cluster-based wave index | 图索引、聚类索引或不使用 ANNS 的轻量打分器；索引建立/更新成本需摊入每个上下文生命周期 |
| 索引和 KV 放在哪 | [ECHO](../sources/echo.md) 用 host memory + GPU cache；[Beluga](../sources/beluga.md) 与 [SAC](../sources/sac.md) 使用 CXL KV pool；[CXL-ANNS](../sources/cxl-anns.md) 把 ANN graph 放在 CXL，并做热邻居缓存、预取和 endpoint 协作 | HBM 放低延迟热数据，host DRAM 作近端容量层，CXL pool 扩展共享容量；索引与 payload 可以分开放置，实际划分仍需测量 |

## 已建立的技术联系

### Query-key 分布差异是注意力 ANNS 的第一道问题

通用 ANNS 常假定 query 与索引向量同分布。RetrievalAttention 发现 attention 的 Q/K 来自不同投影，普通索引可能要扫描大量 keys；它先利用已知 query–key 关系建立桥接，再将关系投影到 key graph。RoarGraph 在 cross-modal ANNS 中讨论相似的 OOD graph traversal 问题，并提供了 projection 思路。两者之间的借鉴有论文直接引用支撑。

RoarGraph 解决的是跨模态向量检索；Q/K 不等于 text/image 两个模态。应把它视作相关的 OOD 索引设计，而不是已验证的注意力替代方案。

### CXL 改变索引和 KV 的位置，不会自动令远程检索变快

CXL-ANNS 的朴素 memory-pool 基线相对 DRAM oracle 出现显著图检索变慢，作者进一步加入 entry-neighborhood caching、候选驱动预取、endpoint distance compute 和更细粒度任务调度。它给 attention index 的启发是：图节点随机访问的 locality、prefetchability、host/CXL/GPU 之间的计算位置必须共同设计。Beluga 的细粒度 sparse-token microbenchmark 也显示 CXL 和 RDMA 在非连续小块访问上的差异，但不是 end-to-end ANNS 或 sparse-attention 测试。

SAC 的结果说明 DSA 可以先在 GPU 选出 top-k，再让 GPU kernel 从 CXL 取回 payload；这避免将 ANN graph traversal 本身放入 CXL。若把 RetrievalAttention 的 ANNS 索引也放进 CXL，就会多出“query → 多跳 index metadata → KV IDs → KV payload”依赖链。CXL-ANNS 不包含 attention Q/K 的 OOD 分布，也没有验证 ANN recall 是否足以保持 attention 输出；三者合在一起仍是跨论文推论。

### Sparse attention 减少读取量，仍会受容量和控制路径约束

DeepSeek-V3.2 的 DSA 将主 attention 限定在少量 KV entries，但 lightning indexer 仍对历史位置打分。ECHO 进一步指出，native sparsity 并不消除完整 KV 生命周期和并发容量压力，于是采用 host offload、GPU cache 和 GPU graph 内 cache manager。SAC 则将 DSA top-k payload 按需放在 CXL pool；但它不使用 ANNS。

RetroInfer 则把 index 与 buffer manager 联合优化：centroids/metadata 驻 GPU，KV blocks 驻 CPU 并聚类访问。这给 CXL 扩展提供可讨论的布局基线，但该论文使用 CPU DRAM + PCIe 测量，没有 CXL 结果。

### 本轮纳入的最强相邻系统收窄了 proposal 的主张

- [HiSparse](../sources/hisparse.md) 已实现 exact、indexer-agnostic 的 sparse selection resolution、固定大小 GPU cache 和按层预取。proposal 不能把“精确搬回被选 KV、bounded HBM residency、跨层 prefetch”单独作为创新。
- [Fluxion](../sources/fluxion.md) 已把 sparse budget、head/granularity 选择和 CPU/GPU attention 调度协同起来；[ScoutAttention](../sources/scoutattention.md) 也有 CPU/GPU co-attention 与异步 recall。异构 attention 与跨设备重叠都已有先例。
- [IceCache](../sources/icecache.md) 已覆盖动态语义页、GQA union 和 bulk backload；[SPIN](../sources/spin.md) 覆盖统一稀疏分页、动态 HBM cache 和 working-set metadata；[CompactAttention](../sources/compactattention.md) 在 chunked prefill 中覆盖 mask-to-page-table 和 GQA/subgroup union。
- [CXL-Vector](../sources/cxl-vector.md) 已覆盖 DRAM 压缩图导航并从 memory-only CXL 拉回原精度候选重排；[CXL-ANNS](../sources/cxl-anns.md) 与 [COSMOS](../sources/cosmos-cxl-anns.md) 分别覆盖 CXL graph search 和 CXL endpoint 内计算。将 RoarGraph/OOD attention queries 用于 KV 与它们组合，仍需要单独实测。
- [TRACE](../sources/trace-cxl.md) 与 [PNM-KV](../sources/pnm-kv.md) 表明 CXL 设备内部的数据表示、压缩、精度层和 near-memory compute 也可能改变瓶颈；它们的硬件假设要与 commodity memory-only CXL 路径分开。

据此，当前 proposal 更可证伪、也更窄的核心问题是：**固定 selector、选集、逐 head mask 和精度后，系统能否按真实 KV group 的物理 footprint、HBM residency、GQA overlap、span 和链路 credits，把冷数据在有界 GPU packet 与 CPU partial attention 之间分流，并在计入 planner、搬运、同步和 merge 后，胜过最强 whole-group 路径？** 这是待验证的研究假设，不是文献已经证明的空白。需要至少和 HiSparse 式精确 GPU fetch/cache、Fluxion/IceCache 式 CPU/GPU whole-group 或 bulk route、SAC 的 CXL 按需读取、以及 CXL-Vector 机制下的候选路径在相同 selection 和质量条件下比较。

selector/质量轴也应单独控制： [Louver](../sources/louver.md) 的零 false negative 是阈值相对保证，不等同固定 top-k 或输出误差界；[Verified vAttention](../sources/vattention-verified.md) 给 sparse attention 近似误差保证；[MiniMax Sparse Attention](../sources/minimax-sparse-attention.md) 使用 per-GQA-group 训练 selector；[Self-Indexing KVCache](../sources/self-indexing-kvcache.md) 与 [SALS](../sources/sals.md) 则探索压缩表示兼作检索空间。它们帮助固定并说明 selection 契约，但不代替同一选集下的执行成本实验。

## 一个可检验的组合架构（推论）

以下是跨论文综合出来的研究假设，不是任何单篇论文已实现的系统：

1. 选择器可复用 DSA-style learned score indexer，也可对固定上下文构建 query-aware Q/K graph 或 wave 类 cluster index；后两者需覆盖真实 attention query 分布以降低 OOD 检索误差。
2. HBM 缓存 learned indexer 必需状态、ANN 顶层入口/centroids、static tokens 和最近访问 KV；host DRAM 可缓存 graph hot-neighbors；CXL pool 放较大的图结构与 KV payload。SAC 证明了 DSA 选择结果到 CXL payload 的一条路径，Beluga 提供共享 CXL pool 的另一种系统实现。
3. 若 ANNS 在 CXL 上执行，需减少依赖型随机读；可评估 CXL-ANNS 的关系感知缓存、预取和近数据距离计算。返回 candidate IDs 后再取回 KV blocks，由 GPU 对候选做 attention，并保留方法要求的 static tokens 或 residual。
4. 用 graph traversal 历史、cache hit/miss 与 query 顺序预取；pipeline 索引遍历、CXL 访问、KV block recall 与 GPU attention。只有当索引额外流量足够小，且候选质量保持 attention 输出时，节省的 KV 流量才可能抵消 index path 成本。

这条路径的关键不是单看 CXL 带宽，而是候选发现的依赖链：query → index metadata/graph traversal → KV IDs → KV payload → attention。若索引图在 CXL 上需要多轮依赖型随机读取，CXL latency 会先于带宽成为瓶颈；若 query distribution 变化导致 ANN recall 下降，系统又可能需要扫描更多 KV。

## 需要用实验回答的问题

- 使用 attention-output error、下游任务质量和 recall/attention mass 一起约束 ANNS；单独报 recall@k 不足以说明 softmax 输出误差。
- 分开报告 index build/update latency、metadata bytes/query、CXL reads/query、KV bytes/token、prefetch accuracy/cache hit、GPU idle time、ITL 与 P99。
- 改变上下文长度、batch/concurrency、每层/head 的 sparsity、query drift、CXL pool topology 与本地 DRAM 容量，测量收益边界。
- 比较 graph ANNS（RetrievalAttention/RoarGraph 路线）、cluster index（RetroInfer 路线）、trained score indexer（DeepSeek/ECHO 路线），避免只与 full attention 比。
- 检查共享 CXL pool 的 multi-tenant contention、ownership、隔离和 tail latency；这连接到 [KV hierarchy survey §3、§5.5](../sources/distributed-kv-hierarchy-survey.md)。

## 建议阅读路径

1. [RetrievalAttention](../sources/retrievalattention.md) → [RoarGraph](../sources/roargraph.md)：attention query-key OOD 与 ANN 图投影。
2. [CXL-ANNS](../sources/cxl-anns.md) → [Beluga](../sources/beluga.md)：CXL 图索引与 CXL KV pool 两种 placement/data path。
3. [DeepSeek-V3.2](../sources/deepseek-v3-2.md) → [SAC](../sources/sac.md) → [ECHO](../sources/echo.md)：native sparse score indexer 到 CXL/host offload。
4. [RetroInfer](../sources/retroinfer.md)：attention-aware index、KV block locality 与异构 buffer manager。
5. 回到 [KV cache 管理与 serving 总览](kv-cache-management.md) 和 [From Tensor Buffer survey](../sources/distributed-kv-hierarchy-survey.md)，把索引路径放回 lifetime、ownership 和 substrate 维度。
