# KV cache 管理与 serving 系统

## 范围

本文关注 decoder LLM 推理时生成并反复读取的 key/value 状态，以及负责分配、搬运、复用、压缩和调度这些状态的系统机制。KV 容量随 resident token 数线性增长；decode 每一步读取已有上下文，长请求会同时放大容量和读取带宽压力。[From Tensor Buffer survey §2](../sources/distributed-kv-hierarchy-survey.md)

## 两种互补的观察坐标

| 坐标 | 维度 | 能回答什么 | 来源 |
|---|---|---|---|
| 优化发生层次 | token-level、model-level、system-level | 方法改变 token 集合、模型/注意力表示，还是 serving 管理？ | [KV cache survey §§III–VI](../sources/survey-kv-cache-management.md) |
| 系统管理语义 | locality、lifetime、ownership、substrate | KV 放在哪里、保留多久、谁管理、怎么访问或搬运？ | [Distributed hierarchy survey §3](../sources/distributed-kv-hierarchy-survey.md) |

综合理解：前一组描述技术落点，后一组描述状态管理契约。比如 token selection 是一种优化机制；它可以运行在本地 HBM，也可以用于 host offload，不能单靠“token-level”推断它的 placement 或 ownership。

## 代表性系统

| 工作 | 主要问题 | 机制 | 设计边界 |
|---|---|---|---|
| [PagedAttention / vLLM](../sources/pagedattention.md), SOSP 2023 | 动态序列长度导致 GPU KV 预留和碎片 | block table 将逻辑 KV blocks 映射到非连续物理 blocks；按需分配并支持共享 | kernel 需要理解分页布局；论文测得 kernel microbenchmark 有额外 block-table 开销 |
| [InfiniGen](../sources/infinigen.md), OSDI 2024 | host offload 每步搬运整份 KV 成本高 | 预测下一层重要 token，只预取所需 KV，冷 token 留在 CPU 或淘汰 | 面向 offloading-based 系统，性能与选择准确度相关 |
| [DistServe](../sources/distserve.md), OSDI 2024 | prefill 和 decode 争用 GPU，TTFT 与 TPOT 目标不同 | 两阶段部署到不同 GPU pool，并按带宽和 SLO 协同放置、并行化 | 产生阶段间 KV 传输；主要关注请求级 handoff |
| [CacheGen](../sources/cachegen.md), SIGCOMM 2024 | 复用远程 context 时 KV 网络传输慢 | 利用 KV 分布特征压缩，并随带宽调整编码/重算选择 | 优化传输表示，不提供全局 KV ownership 或长期淘汰策略 |
| [vAttention](../sources/vattention.md), ASPLOS 2025 | 应用层分页使 kernel 显式处理非连续 KV | 用 CUDA VMM 分离虚拟与物理分配，保持虚拟地址连续 | 依赖 VMM API 与平台支持；与 block paging 是不同抽象 |
| [Mooncake](../sources/mooncake.md), FAST 2025 | 长 context 与重复 prefix 造成重复 prefill、GPU KV 容量受限 | P/D 解耦、全局 KV store、CPU/DRAM/SSD/RDMA、多级放置和调度 | 收益依赖复用率、存储资源、互连及 workload/SLO |
| [FlashInfer](../sources/flashinfer.md), MLSys 2025 | KV layout 和 attention 变体多，专用 kernel 难维护且负载不均 | block-sparse/composable layout、JIT attention template、动态调度 | 解决 kernel/runtime 数据路径，不定义 KV 生命周期或全局控制面 |

## 跨论文综合

### KV ownership 和数据位置是不同选择

KV 可以放在 GPU HBM、host DRAM、SSD、远程节点或 memory pool；这些位置并不决定由哪个组件命名、分配、淘汰和仲裁访问。分布式系统可能用 coordinator，也可能用分布式目录。这个区别影响控制路径和故障语义，是 [From Tensor Buffer survey §3.3](../sources/distributed-kv-hierarchy-survey.md) 提出的核心轴之一。

### P/D 解耦有两种不同规模

[DistServe](../sources/distserve.md) 通过跨 GPU pool 的请求级 KV handoff 分离计算阶段；[Mooncake](../sources/mooncake.md) 在此基础上增加跨请求 prefix reuse、全局缓存和多层存储。两者同属 disaggregation 方向，但 lifetime 与 reuse scope 不同。[这是跨论文综合，分类依据见综述 §4](../sources/distributed-kv-hierarchy-survey.md)。

### 控制面、数据面和 kernel 必须一起看

PagedAttention 和 vAttention 对比 allocator/地址空间抽象；FlashInfer 关注 kernel 如何消费分页、稀疏和变长 layout；DistServe、Mooncake 与 CacheGen 关注 KV 如何跨阶段、请求或节点移动。把这些工作统称为“KV cache optimization”会掩盖它们分别优化的路径。

### 论文数字还不能组成统一排行榜

不同论文分别报告 throughput、SLO goodput、KV 压缩率、request capacity、kernel latency 或质量。模型、硬件、网络、trace 和基线各异。2026 survey §5.5 指出，remote access pattern、metadata path cost、completion overhead、lifetime/reuse distance、prefetchability、P99 attribution 和公开 trace 尤其缺乏。

## 后续值得追问

- 对一个实际 workload，prefix reuse 的命中率与 KV transfer bytes 到什么程度时，Mooncake 类 store 能胜过本地 prefill？
- 在相同 GPU 和 kernels 上，PagedAttention 与 VMM-based allocation 的 allocator 成本、kernel 成本、碎片和 tail latency 如何比较？
- CacheGen 的压缩/重算策略能否作为 DistServe 或 Mooncake 的数据路径组件？需要报告压缩质量、端到端 TTFT、互连占用及 decode 影响。
- 按 From Tensor Buffer survey 的建议，能否用同一公开 trace 同时报出 reuse distance、远程访问模式、metadata 成本与 P99 归因？
- sparse selection、ANNS 索引和 CXL tier 如何组合，以及远端图遍历开销是否会抵消 KV 流量下降，见 [Sparse attention、CXL 与 ANNS 专题](sparse-attention-cxl-anns.md)。
