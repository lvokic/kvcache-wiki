# CXL-KV Proposal v2.1：OOD 向量检索与三层 KV attention 的协同执行

更新：2026-09-29。工作题目：**OOD-Aware Retrieval and Physical Planning for Attention over GPU, DRAM, and CXL Memory**。

范围：长上下文自回归 decode、训练后稀疏 attention、单节点 GPU + CPU/DRAM + memory-only CXL。本文为研究设计，没有模型、GPU kernel 或 CXL 性能实验结果。文献事实、分析示例和待验证机制分别注明。配套《CXL_KV_Architecture_v2.html》给出部署图、单层执行 DAG、数据契约和执行示例。

## 1. 此次修改的实质

v2.1 补齐向量检索：IVF-Flat/SQ8 为基线，原精度 RoarGraph 隔离拓扑收益，CXL-Vector + RoarGraph 为分层执行候选。新增第7节给出构建、查询、GQA、原 K 重排、K lease、追加协议、容量和否定实验。检索侧主问题是 OOD 导航质量与 CXL 原 K 读取的权衡；执行侧继续研究固定选集下的物理规划。二者需分别验证，不预先把所有模块包装成创新。

新增研究笔记明确聚焦 **kernel × index × data layout**。[U1] 据此，v1 的“质量需求与共享带宽尾延迟”降为约束和扩展实验，主贡献改为：

> 在选择器已经确定所需 attention 集合后，如何利用真实 GQA 重叠、物理布局、驻留状态和设备负载，把同一集合转换成更便宜的读取、传输与计算计划？

建议优先研究 **按物理数据片段进行 late materialization 和 CPU/GPU 分流**：检索结束后按 KV group 合并物理读取，保留每个 Q head 的逻辑掩码；GPU 常驻 KV 原地计算；适合批量搬运的冷数据形成有界紧凑 packet；零散或搬运不划算的部分在 CPU 计算 partial attention。同一个 head 可以由多条路径共同完成。

“CPU/GPU 合作”“partial merge”“GQA union”“packing”均不是新概念。待验证贡献是：**选择完成后，以真实物理 footprint 为依据，在同一 KV group 内做有成本约束的分流，并提供足够便宜的执行接口。** 固定选择器、选集和精度，系统只改变位置、顺序和中间表示。若没有净收益，就退回 whole-group 路径。

| 原版的缺口 | 新版落实到的设计 |
|---|---|
| KV manager 是黑盒 | 逻辑选集、版本目录、执行描述符、数据通路、完成归并各有契约 |
| 默认“取回再 attention” | HBM 原地、CPU partial、pack→GPU；mapped access 是可选 backend |
| 粒度只有概念 | page、extent、packet、tile 独立配置 |
| layer policy 主导架构 | 先完成单层执行链；层策略只决定可接受选集与 full 模式 |
| CXL tail 是主要动机 | 先查重复读取、物化、同步；再查 CXL 对路径胜负的影响 |
| PP 只是图上的模块 | 明确真实 Q 依赖、地址可知时间、stage 边界与共享 credits |

## 2. 进一步研究后的创新边界

以下依据论文正文与官方接口；不是对所有实现的逐行代码审计。

| 来源 | 已有能力 | 本方案必须超越的部分 |
|---|---|---|
| IceCache，ICLR 2026，§4 [R1] | 动态 DCI、语义页、key IDs→pages、GQA union、CPU backload buffer→GPU buffer→scatter | 图+page、动态更新和 bulk loading 均不新；不 scatter 应成为简单强基线 |
| FlashInfer，§3、App. A/B/D [R2] | 稀疏行加载到 shared memory、GQA fusion、split-KV、plan/run、state merge | 不重写通用内核；其测试中 decode sparse gather 与连续路径差距很小，不能预设稀疏 GPU 访问一定低效 |
| SPIN，§4–5 [R3] | head-wise page、分层元数据、动态 HBM、persistent warp copy | 泛化存储 API、分页 offload 与高效 copy 已有覆盖 |
| Fluxion，§5–6 [R4] | 预算/粒度预测；一个 GQA group 的 selection+attention 为任务，CPU/GPU 共享队列 | 比较点是 selection 后的真实 footprint 与 group 内分流，不是首次异构调度 |
| RetrievalAttention，正式版 [R5] | CPU 历史检索/attention 与 GPU 常驻部分合作 | 同一 head 的跨设备分解已有先例；需胜过固定切分 |
| Strata，OSDI 2026，§4 [R6] | host/GPU 布局解耦、GPU-assisted I/O、cache-aware serving | 布局转换/重叠本身不新；大范围 context loading 与逐层 decode 稀疏读要区分 |
| DirectKV，OSDI 2026，§4–5 [R7] | NVLink-C2C 直接访问 CPU KV、tiling、warp pipeline；包括 prefill 的 projection/attention fusion | GH200 强互连上的收益不能外推到 PCIe+CXL |
| Swarm，§5–6 [R8] | co-activation 聚类、多 SSD 放置、在线适应 | co-access layout 不是空白；先固定布局验证执行空间 |
| CompactAttention，§3、App. B [R9] | chunked prefill 的 GQA/subgroup block union、head-major 元数据与原地 paged attention | mask→page table、免 compaction 已有先例；coverage preservation 不等于逐 head 语义等价 |
| SAC / Beluga [R10–11] | CXL KV 获取、真实 CPU/GPU 路径及拓扑瓶颈 | 不声称首次 CXL sparse KV；设备、root complex、并发都要测 |

一个会直接影响代码的发现：当前 FlashInfer 的部分 `block_mask` backend 要求同一 KV group 的 Q heads 共用稀疏模式；`VariableBlockSparseAttentionWrapper` 也明确写出这一约束。[R12] 因此不能简单“做 union 后调用现成 kernel”。

实现区分 **group-shared selection** 与 **head-specific selection**。前者可复用适配的 paged backend；后者需要支持 head mask 的 adapter/kernel，或将 head 作为独立逻辑行。独立逻辑行可以引用相同 payload，但可能损失硬件 KV 复用，必须测量。API 以锁定 commit/backend 为准。

## 3. 问题定义：逻辑稀疏不等于物理便宜

固定 request、decode step、layer、KV group。r 个 Q heads 的最终选集分别为 S_h，包含规定的 sink/window/delta 与历史选集。定义：

\[
U=\bigcup_{h=1}^{r}S_h,\qquad M_{h,i}=\mathbf 1[i\in S_h].
\]

U 决定可以合并的物理 KV；M 决定 attention 语义。让所有 heads 都 attend U 会改变 S_h；即使覆盖更多 tokens，也不能称为原选择器的等价执行。

至少区分：逻辑选择量 Σ|S_h|；去重后 |U| 与 HBM 缺失量 |U_miss|；CXL/DRAM/H2D 各自 bytes/transactions；实际执行 tiles、padding、mask 和重复加载。

K/V 维度均为 d、每元素 b bytes 时：

\[
B_{use}=2db|U_{miss}|,\quad R_{share}=\frac{\sum_h|S_h|}{|U|},\quad
\rho_{mask}=\frac{\sum_h|S_h|}{r|U|}.
\]

R_share 表示可复用程度；ρ_mask 是完整 union tile 中有效 head-token 对比例，不等于 GPU 实际利用率。另报 A_CXL=B_CXL/B_use、A_H2D=B_H2D/B_use；索引/摘要流量单列，不能只计返回 token。

**分析示例，非实测。** r=4，每 head 选 1024 tokens，d=128，BF16。若 union=1536，唯一 K+V 为 768 KiB；若互不重叠，union=4096，为 2 MiB。逻辑预算相同，ρ_mask 分别为 2/3、1/4。第二种情况下，若每个选中 token 分散在不同的 16-token page，整页加载为 32 MiB；slice gather 的有效 payload 仍是 2 MiB。16× 是整页策略在此构造下的代价，不是不可避免的 CXL 放大。

核心可证伪假设：**固定 token budget，甚至固定 unique bytes，仍不足以预测最快路径；地址分散度、head 共享、驻留和同步次数可能改变 CPU/GPU 的胜负。**

## 4. 场景与三项不变量

GPU OOM 只证明需要 offload。CXL 场景还需要 DRAM 容量/预算受限，或 DRAM 必须留给索引、热数据和 staging。主实验先用权重能驻留的 GQA 模型，不同时混入权重 offload、MoE 或训练改模。

1. **选集不变。** 物理规划不删减/扩张 M。整页读入的多余 tokens 必须被 mask；若原算法定义 group union，则以该 union 为语义基准并标注。
2. **每个逻辑贡献一次。** 对每个 head，GPU hot、GPU packet、CPU residual、full streaming 分区两两不交，并集恰好为 S_h。
3. **版本一致。** Q、K、position、mask、精度、RoPE/score transform 和索引水位来自同一 snapshot；旧读者结束前不回收对应页。

固定选集的等价执行允许浮点归约差异，不承诺 bitwise identical，更不等于 sparse 与 full attention 等价。质量仍需独立任务评估。

容量分析例（非实测）：32层、8 KV heads、d=128、BF16、128K tokens 的完整 KV 为16 GiB/请求。仅把两层完整常驻就需1 GiB/请求；batch=16 时占16 GiB。这解释了为什么 full 层驻留、packet 双缓冲、索引摘要和 batch 扩大必须共用一个真实显存预算。最终预算分别扣除权重、模型工作区、KV cache、packet ring、partial states 和 graph workspace，不能把全部剩余显存都分给热 KV。

## 5. 系统组件和明确职责

| 组件 | 持有内容 | 产物 |
|---|---|---|
| GPU model executor | 权重、真实 Q、新 K/V、sink/window、HBM cache、定长 workspace | Q-ready event、local partial、append record |
| CPU retrieval / selector | IVF 或 Roar topology、DRAM codes、per-head 搜索状态、原 K 精排 | 最终 S_h、epoch、selected K lease、可选真实 logits |
| CPU physical planner（待验证主机制） | 实际 union/mask、directory、路径 profile、队列负载 | route descriptors、packet plan、head 完成计数 |
| CPU executor / gather | SIMD attention、NUMA-bound gather、pinned ring | CPU partial，或 packet-ready event |
| GPU attention adapter | page list、packet spans、mask、position、split plan | 各 head 的 partial states |
| CXL store | sealed KV extents、追加后备、可选冷索引 | memory bytes；不执行 ANN/attention |
| 生命周期管理 | epoch/refcount、copy/index 水位、容量 credits | snapshot publication 与安全回收 |
| 测量/准入 | 链路、CPU/GPU 负载与 buffer credits | 有界提交、回压与维护保底 |

主路径是 CXL→CPU gather/DRAM staging→PCIe→GPU，或 CPU 原地计算后回传小状态。GPU mapped CXL 是独立可选 backend，先验证 DAX/pinning/IOMMU/driver/coherence 与拓扑；不能画成 GPU 天然具有 CXL.mem 端口。普通 Type-3 内存没有 near-memory compute。

## 6. 数据布局与生命周期

### 6.1 Canonical layout 与四种粒度

每个 `(request, layer, KV group, segment)` 有独立 K array 和 V array，各 token vector 连续、对齐。K/V 分开使索引可以只读 K，不因候选重排自动加载 V。目录含 tier、K/V 地址、valid length、原始 position 映射和 epoch。全历史保留，HBM/DRAM cache 是可撤销副本。

初始 backing page=16 tokens，再扫 16/32/64/128；不是最优值声明。d=128、BF16 时，一个 KV head 的 page 为 K 4 KiB+V 4 KiB。OS page 与 CXL transaction 独立。语义布局作为实验变量，MVP 不在线全局重排。

| 单位 | 用途 | 决定逻辑集合？ |
|---|---|---|
| token/cluster/selector page | 检索选择 | 是，按算法契约 |
| backing page | 分配、映射、缓存 | 否 |
| extent | 同一 tier/epoch 下物理连续读范围 | 否 |
| packet | 有界、紧凑的传输 payload，可组合多个 extent | 否 |
| GPU tile / SIMD batch | 运算与调度 | 否 |

packet 可装多个请求/group 来摊销搬运，但它们仍是独立任务；不能把无共享 KV 的请求当作同一 GEMM 的 query 行来虚构复用。

### 6.2 执行契约

```text
Selection: req, step, layer, kv_group, epoch, semantics, head_indptr, token_ids,
           selected_K_lease, optional_exact_logits
Extent:    tier, K_base, V_base, valid_len, position_map, epoch
Work:      group_ref, route, span_or_offsets, mask_ref, position_ref,
           packet_slot, tile_range, completion_slot
Packet:    K_spans, V_spans, offsets, head_masks, valid_counts, ready_event
Partial:   req, step, layer, head, partition_id, epoch, state_format, state
```

实际实现采用 SoA、offsets/bitmaps，避免每 token 一个 heap 对象。head membership 可用 r-bit bitmap 或 CSR。GPU 只接收 active working set 的 descriptors，不分配全历史 dense mask。`request-local page ID`、`physical token index`、`packet offset` 是不同类型；官方 TensorRT-LLM 接口也区分这些布局。[R13]

索引容量同样需要预算。示例：32层×8 KV groups×128K tokens×32条邻边×4-byte ID，仅邻居 ID 约4 GiB/请求，尚不含向量、offsets 和 allocator。不能无条件假设“所有索引都在 DRAM”。比较 per-group graph、页摘要、分段聚类和热导航分层；是否共享索引结构必须验证检索质量。

### 6.3 Append 与 publication

`GPU_VISIBLE → COPY_PENDING → BACKING_READY → INDEX_READY → PUBLISHED`。

snapshot 使用已发布索引覆盖的前缀，加上精确可访问的 delta 后缀。delta 尚未入索引也必须被处理；sink/window/delta 重叠去重。backing 和索引均就绪才推进共同水位。维护跟不上时，受限扩大 delta、降低准入或同步推进，不能丢 KV。

sealed segments 只读；rebuild 写新版本后原子发布目录，旧版按 epoch 回收。这是可见性协议，不是 crash durability，CXL 内存不自动具有持久性。

## 7. 向量检索模块：IVF、RoarGraph 与 CXL-Vector 分层执行

### 7.1 选型结论与层次

**优先实现 IVF，保留原精度 RoarGraph 作对照，把 CXL-Vector + RoarGraph 作为主研究候选。** 这不是三个完全互斥的算法：IVF 与 RoarGraph 决定检索拓扑；CXL-Vector 提供压缩导航、原向量分层和批量执行机制。以下组合是本 proposal 的设计，不能当作 CXL-Vector 稿件已有的 RoarGraph 实现。所给稿件实际研究 HNSW/NSG，并使用 SQ8/TQ5。[U2-C]

| 路线 | DRAM 常驻内容 | 原精度 K 的访问 | 角色与主要风险 |
|---|---|---|---|
| A1：IVF-Flat / IP | centroids、list IDs；K 放置单独配置 | 扫描被探测 lists 的 K | 正确可测的起点；顺序 SIMD 较强，但高 nprobe 会扩大读取 |
| A2：IVF-SQ8 + 原精度重排 | centroids、连续 codes/IDs | 只精排保留候选 | 必须具备的强基线；与组合路线使用相同原 K/V 存储和重排后端 |
| B：RoarGraph 原精度导航 | projected key graph；原 K 的 DRAM/CXL 放置分开测 | 遍历访问原 K | 隔离 OOD 拓扑收益；原 K 全在 DRAM 是容量较大的延迟参照，原 K 在 CXL 暴露随机读代价 |
| C：RoarGraph + CXL-Vector 机制 | projected key graph、SQ8 或 TQ5 codes | DRAM 导航后，从 CXL/缓存取有限候选 K 重排 | 主候选；需证明压缩后拓扑收益仍在，且图、codes、临时状态装得下 |

先按 layer/KV group 的校准配置选择一种历史索引，不同时为每个 group 长期维护三套索引，也不在每次 query 间无成本切换。不能稀疏化的层绕过 ANN，执行已定义的 full 路径；“稀疏成立但 ANN 不划算”的层可以 scan。IVF 不必然因 OOD 失败；较大的 nprobe、连续读取和批量计算可能仍胜过图搜索。

### 7.2 检索目标、GQA 与误差分解

检索使用模型真实位置处理后的 K，以及当前真实 Q。标准目标为最大化 `q_h^T k_i / sqrt(d)`；若模型另有 score transform、bias 或 mask，需要保持实际语义或单独限定适用模型。不能将 K 单位归一化后用 cosine 代替原始 MIPS；这会改变排名。[R16]

本节 k 是历史检索预算；历史结果记为 T_h，完整逻辑选集仍为 S_h=T_h∪mandatory_h。U_k 指历史 T_h 的 union，mandatory 的 attention 在独立路径处理并去重。

建议每个 `(request, layer, KV group, sealed segment)` 维护一套历史索引。该 group 的各 Q heads 共用 K、codes 和图，**独立保留搜索状态、候选集和最终 top-k**。共用图是本方案需要验证的设计选择；RoarGraph 建图 Q 样本应覆盖组内各 heads，并比较 per-head 图的质量/容量参照。batch 不同请求没有共同 K，不虚构跨请求的物理复用。

至少拆开三类误差：full attention→exact top-k 的截断误差；exact top-k→原精度 ANN 的路由误差；原精度 ANN→压缩导航的额外误差。最终“原精度重排”仅保证已保留候选的分数按原 K 重新计算，不能恢复从未进入候选池的 key，也不保证 sparse 与 full attention 等价。

### 7.3 路线 A：具体的 IVF 模块

以 Faiss `IndexIVFFlat`、`IndexIVFScalarQuantizer` 与 inner-product metric 为实现参照；实际 serving adapter 独立管理 canonical KV、epochs 和缓存，不能假定通用 Faiss 对象已经支持这些契约。[R17]

1. **构建。** 对当前索引覆盖的 K 训练 coarse centroids；训练只用当前可用数据。采用 IP-compatible 的训练/assignment 配置，并记录是否使用 residual encoding。centroids 可以是归一化的路由代表，但不得因此归一化原 K、改变 attention 分数。跨请求共享 centroids 是单独实验，不能省略其质量代价。
2. **布局。** DRAM 中按 list 连续存储 `codes + stable token IDs`，list position 经目录映射到 canonical K/V。先固定原 K/V 的 token-major 布局；将原 K/V 也按 list 重排作为独立布局消融，避免混淆索引与布局贡献。
3. **查询。** 每 head 探测 nprobe 个 lists。Flat 对这些 lists 中的原 K 计算真实 IP；SQ8 用压缩分数形成至多 R 个候选，再调用共同重排模块。Flat 已经取得的 K/真实分数可直接复用，不强制再次精排或读取。
4. **追加。** 已训练 centroid 下，新 K 可以分批 assignment、编码、加入新版本的 list blocks；查询中的旧 blocks 不原地改写。coarse model 漂移后重训、重编码和发布均需计成本。

128K 的探索网格可以从 `nlist ∈ {64,256,1024}`、若干 `nprobe ≤ nlist` 开始，同时检查训练样本量与 list imbalance；这些是实验起点，不是已验证最优值。nprobe=nlist 的 Flat 路径用于正确性参照。SQ8 的候选不足 k 时须扩大探测或明确回到约定路径，不能静默返回较少 tokens。

### 7.4 路线 B：把 RoarGraph 落实到 attention

RoarGraph 的关键是用 query–base 近邻关系构造二部图，再经 neighborhood-aware projection 与连通性处理形成 **key-only 的搜索图**。在线搜索不是带着 Q 节点遍历二部图。官方代码支持 inner product；RetrievalAttention 已使用相关投影思路，因此“将 RoarGraph 用于 attention”本身不是本文创新。[R15][R5]

**Q 样本来源。** 从当前可见 prefill 或历史已发生 decode 中抽样，按 layer/KV group/Q head/position 分层。新请求可以使用其 prefill Q；不能使用未来 decode Q。prompt 内 Q 与后来 decode Q 的分布差异仍然存在，必须在 held-out decode、multi-turn 和长生成上测试。保存 RoPE 后的真实表示及版本。

**建图成本。** 对 Q 样本求建图所需的高质量近邻，再投影/剪枝/修复。若采用精确 streaming top-N，成本近似随 `|Q_train| × N × d` 增长。FlashAttention 不物化完整 QK，不能声称 prefill 已“免费产生”所需邻接关系。可用 causal-valid 的 prefill Q–K 对作初始构建方案，并测试其对后部 K 的覆盖；降低 Q 采样或近似构建分别记录质量损失。原论文将 ground-truth 生成视作主要构建开销，官方实现也提醒这一阶段可能很慢。[R15]

**原精度参照。** 固定一份建好的 topology，分别运行原 K 导航和压缩 K 导航。原 K 全 DRAM 与原 K 在 CXL 两种配置分开报告；不把改变放置带来的收益算成 RoarGraph 的拓扑收益。每 head 用独立 beam/heap 和 exact visited，GQA 可以合并读取，但不合并搜索语义。

**追加边界。** 原论文的插入实验是 offline insertion，保留 query–base 二部信息并更新图，不能据此假定最终 key-only 图天然支持廉价逐 token 在线更新。[R15] MVP 采用只读 base + 精确可见 delta；具体发布协议见7.7。

### 7.5 路线 C：CXL-Vector + RoarGraph 的执行内核

复用所给稿件的机制，而不是照搬其工作负载参数：[U2-C]

- **放置。** RoarGraph 的 adjacency 和一份 compact K code 留在 DRAM；原精度 K/V 分离存于 CXL，热副本可在 DRAM/HBM。导航只读 adjacency/codes；不沿图随机拉取 V。
- **导航内核。** 将 neighbor collection、visited filtering 与 distance evaluation 分开，积累一批候选后做 SIMD。批内更新 admission threshold，不能用旧阈值错误丢弃候选。预取下一批 codes/邻接表；收益要在 d≈128 的 attention 维度重新测。
- **编码。** 首先 SQ8，随后 TQ5；所给稿件不是默认 PQ 方案。编码/反编码、query quantization、TQ5 lookup/requantization 引入的排序误差均计入对照；不能用代码距离直接作为 softmax logits。
- **边界。** 若 graph+codes 放不下 DRAM，就已越过本设计的假设；可以研究冷导航分层，但必须将其 CXL 读入总成本，不能仍画成“只有最后重排访问 CXL”。
- **visited。** 初版使用 exact bitset 或带 generation 的状态；128K 节点 bitset 为16 KiB/活跃搜索，按 worker 复用。稿件的 blocked Bloom 优化具有 false-positive skip 风险，作为有质量标注的消融，不默认继承。最终候选去重必须精确。huge-page 优化基于池化分配，避免给每个小索引浪费独立大页。

搜索维护容量 W 的保留池，从中取 R 个候选精排，最终返回 k 个历史 tokens：`W ≥ R ≥ k`，均不超过该分段有效基数；visited 数可以超过 W。实验扫 `R/k ∈ {1,2,4,8}` 并联合调 W、图 degree 和构建预算。CXL-Vector 在常规小 top-k 向量搜索上的重排参数，不能直接移植到 attention 的 k=数百至数千。

### 7.6 共同后端：候选 K 精排与 attention 的所有权交接

这是对 v2 架构最实质的修正：**CXL 流量在最终选集出现之前就可能发生，检索与 attention 必须共享数据和带宽预算。**

```text
SearchInput  { req, step, layer, group, epoch, Q_heads,
               scale_and_position_semantics, mandatory_ids, index_config }
Candidates   { head_indptr, token_ids, approximate_scores, epoch,
               optional_exact_score_ref, optional_K_lease }
KRefine      { unique_candidate_ids, per_head_membership,
               K_locations, buffer_credits, ready_event }
Selection    { final_head_indptr, final_token_ids, semantics, epoch,
               selected_K_lease, optional_exact_logits }
KLease       { owner, epoch, token_to_buffer_offsets, precision,
               ready_event, remaining_consumers }
```

**具体执行：** 各 head 的 R 候选形成 `C_h`；对同 KV group 去重得到 `U_R=∪C_h`。按 cache/directory 解析 K 的位置，CXL miss 才取原 K，批量精排；每 head 只在自己的 C_h 内选历史 top-k。候选 union 只是读取优化，不能偷偷把其他 head 的候选加入自己的精排集合。排除 mandatory 集合时应在检索过滤/预算中处理，保证约定历史预算而非事后删掉后不足 k。

最终历史 union 为 `U_k`。精排后立即释放未选中候选的 K，选中 K 以 lease 交给 physical planner。后续 CPU partial 或 GPU packet 优先直接使用这些 K，**再获取缺失的 V**；避免“精排读 K，一般 KV gather 再读同一个 K”。GPU 已有 KV 的情况优先用其常驻路径，跨设备回读 K 需要单独权衡，不能假定 HBM 对 CPU 免费可见。

K buffer 的内容键至少包含 request/layer/group/epoch/token；logit 缓存还必须包含 step/head/score semantics。lease 在所有读取/H2D 消费者结束后释放，容量不足触发有界回压；若允许提前驱逐，就把后续重读计入成本。CPU 可复用精排的真实 logits；GPU adapter 是否消费预算 logits 作为可选实验，不为了省 d=128 的 dot product 强制增加传输。

忽略 cache-line/page 放大、导航溢出与写入流量，在无重复读且原 K/V 分离时，历史路径的有效冷读 payload 为：

\[
B_{CXL,payload}=d_Kb_K|U_{R,coldK}|+d_Vb_V|U_{k,coldV}|.
\]

这里第一个集合是**候选 K**，第二个才是**最终 V**；两者可以有不同命中率。三层缓存、graph/code 的额外冷读、事务取整、维护写入另加。若 selected K lease 丢失，公式还需加其重读量。假设无命中、跨 head 无重叠且维度/精度相同，则相对只读最终 K+V 的 payload 放大为 `(R/k+1)/2`；R=4k 时为2.5倍。这说明 RoarGraph 的访问节点少不等于 attention 的 CXL 流量小。

所有 refine reads、最终 V reads、full-layer streaming 和 append 共用链路 credits；给正在持有 buffer 的任务保留完成所需资源，避免 buffer 与 read credits 相互等待。阶段队列要有明确公平性和维护保底。PP 的两个 stages 共用这组真实链路预算，不能各自按独占 CXL 峰值估算。

### 7.7 构建、delta 与版本推进

**新请求。** 完成必要 prefill 后即可先用可靠 scan/IVF 路径服务，RoarGraph 如异步构建则记录争用、切换和请求生命周期总成本；若在首 token 前同步建图，全部计入 TTFT。切换前后必须沿用同一已校准质量目标，不能将低质量热身当作免费阶段。可复用前缀的 graph/codes 仅在模型、层、位置处理和 K 完全匹配时复用。

**追加。** 新 KV 先进入精确可见 delta；尚未发布的部分是 mandatory，可直接 attention，不能因未入 ANN 而漏读。delta 达阈值后封存一段，完成 backing、codes 和索引才原子发布；期间保持旧 snapshot+delta。IVF 可采用新 list blocks；Roar 路线先做 immutable segment build，不假设逐 token 图插入已解决。

**分段代价。** 多段查询需要搜索各段再合并真实分数。若各段都做 exact local top-k，则它们的并集包含 global exact top-k；ANN 局部检索仍有自己的误差。为了减少开销而每段只取 k/段数不能保证上述性质。随着段数增长，搜索和 rerank fan-out 必须计时；设置段数上限和 compaction 预算，避免用无限分段隐藏维护成本。重建可能需要旧版+新版双份容量及 query 样本/二部边临时空间。

**资源不足。** 可以减小服务并发、暂停准入或同步推进维护；不能为了满足 bytes 上限悄悄降低 k、丢 delta 或返回不完整分区。query 漂移检测先作为记录和离线重建触发，不能把未校准的在线质量保证写成事实。

### 7.8 容量与 build amortization：为何不能预先宣布图赢

以下为分析例，非测量：32 layers、8 KV groups、128K tokens、d=128、BF16，若全部 groups 建索引，共33,554,432个 key vectors。假设每节点32个4-byte邻居 ID，无额外 padding：

| 内容 | 容量 |
|---|---:|
| 原 K / 原 V | 各8 GiB；完整 KV 共16 GiB |
| 图邻居 ID | 4 GiB |
| SQ8 K codes | 4 GiB |
| TQ5 K codes，理想紧凑存储 | 2.5 GiB，实际含 codebook/padding 更高 |
| 原精度 Roar 的 DRAM graph+K | 至少12 GiB |
| 分层 Roar 的 DRAM graph+SQ8 | 至少8 GiB |
| 分层 Roar 的 DRAM graph+TQ5 | 至少6.5 GiB |

因此 SQ8 组合相对原 K 常驻 DRAM 的 graph+K，示例中只节省约1/3 DRAM，而非数量级。它另需 CXL 原 K/V 与临时 DRAM buffers；是否删除重复后备副本须按实际配置记账。IVF 不承担逐节点 degree 个边的成本，但有 list IDs、centroids 和 codes。Faiss 原生 list IDs 通常为8 bytes；若采用本地32-bit ID，是自定义表示，不能混报容量。[R17] 仅为适合 retrieval 的层建图可以按覆盖比例降低开销，但 layer 选择本身受质量约束。

判断建图是否值得的分析条件是：`可复用的 decode queries × 每次净节省 > 相对基线的额外构建/维护时间`；这只是工作量摊销估计，TTFT 和 p99 仍是独立约束。新前缀短输出优先验证 IVF；长 decode、可复用长前缀及 DRAM 受限场景更值得投入组合路线。报告 cold start 与 warm-prefix 两组结果。

### 7.9 更有研究价值的问题：OOD 拓扑收益能否穿过量化与 CXL 重排预算？

建议把检索侧问题收敛为：**在 attention 的 Q/K 分布差异下，怎样分配导航宽度 W 与原 K 重排预算 R，才能以给定质量最小化真正的 CXL 读取和端到端时间？** Roar 的拓扑可能有优势，但低精度导航可能恰好破坏 OOD query 的排序；更大的 R 也可能把节省重新花在 CXL 上。两者需要联合研究。

诊断可从 K 编码误差 `e=k−k_hat` 开始。对未量化 Q，标准 scaled IP 的误差为 `δs=q^T e/sqrt(d)`；若 e 固定，则 `E[δs²]=e^T E[qq^T]e/d`。这使用 query 的二阶矩，并不一般等于 covariance。K 的重建 MSE 小，不代表 attention query 方向上的分数误差小。若 Q 也量化，还需加入其误差项；不能用这个简化式解释全部 SQ8/TQ5 实现误差。

先固定 Roar topology，比较原精度、SQ8、TQ5 在相同搜索预算下的 recall/mass 和 margin flips，再比较达到相同任务质量所需的 W、R、实际 CXL bytes。query-aware 量化/残差保护已有相关方向，不能仅凭上述公式声称新算法；本项目可验证是否对少量敏感维度/节点保留更高精度，比全局增大 R 更便宜。若没有净空间，保持通用编码即可。

**实验矩阵。** IVF-Flat、IVF-SQ8、原精度 Roar、Roar-SQ8、Roar-TQ5；同模型、数据、任务质量、canonical layout、cache 容量与执行 backend。单独消融 K lease、跨 head 候选去重、batched traversal 和原 KV 布局。记录 candidate recall、attention mass/error、自由生成质量、建图/维护、DRAM peak、visited/edges、K-refine bytes、final-V bytes、重复 K bytes、TTFT/ITL/p99。拓扑、压缩、放置、执行四个因素逐一分离。

实现来源固定到论文对应 commit：RoarGraph 官方实现可作构建/搜索参照；RetrievalAttention 官方仓库当前 main 已展示 RetroInfer 内容，不能直接将 main 的运行结果标作原论文复现。[R18] 本轮没有实际编译、运行上述索引，也没有测得组合路线的性能优势。


## 8. 单层执行协议

1. **QKV projection 与位置处理。** 产生真实 Q、新 K/V。Q 发往 CPU selector；GPU 同时计算必选 sink/window/delta。GPU selector 可省去 Q 传输。
2. **历史 selection。** CPU 在同 epoch 执行已配置的 IVF/Roar 或 scan/page selector；压缩导航先产出候选，再以原 K 精排，保留 selected K lease。必选集合在选择契约中排除重复。节点、边、候选 K、摘要和精排 CXL 流量均计时。
3. **Finalize→plan。** 按 group 去重，保留 M，解析驻留目录与 footprint，包括精排已读取 K 的 lease。各 group 一旦独立完成 selection 即可规划，不等待整个 layer；普通 ANN 未完成的候选只可做可撤销预取，不能直接成为最终贡献。
4. **三条基础路径并行。** HBM selected KV 原地计算；冷数据复用 selected K、按需读取 V，走 CPU partial 或 pinned pack→H2D→GPU partial。packet 直接被 attention 消费，不强制二次 scatter 到完整 KV cache。若要 cache install，单独记账且不重复计算。
5. **join/merge。** 每 head 收齐所有被计划 partition，检查 completion/epoch，归并统计量。缺必要分区时不能进入 O projection。
6. **O projection、residual/MLP→下一层。** 下一层真实 Q 必须等待前驱计算。
7. **后台 append/maintenance。** 独立 credits 保证推进；它们与读共享资源，须计入排队与带宽。

packet ring 状态：`FREE → FILLING → H2D → COMPUTING → FREE`。存在异步 cache install 时，最后一个消费者完成后才能释放。没有 slot 就回压，不临时无限分配。full streaming 也采用分块双缓冲，无需完整历史同时进入 HBM。

## 9. 核心机制：selection 后的物理规划

### 9.1 候选动作与适用条件

| 动作 | 路径 | 可能适用 | 可能失败 |
|---|---|---|---|
| G_LOCAL | HBM 原地→GPU partial | 已驻留、复用高 | 任务过小、head mask 不兼容 |
| C_PARTIAL | DRAM/CXL→CPU attention→小状态 | 零散冷数据、CPU 空闲、H2D/物化昂贵 | CPU 饱和、CXL 随机读慢、join stall |
| G_PACK | gather/pack→H2D→GPU partial | 可合并的大任务、GPU 更合算 | host 双重流量、等待 packet、copy/compute 争资源 |
| G_MAPPED（可选） | GPU 经验证的映射读 host/CXL | 控制开销低、路径足够快 | SM 等远端访问、RC/PCIe 限速 |
| FULL_STREAM | 连续 extent 分块→partial | 质量要求 full；或 sparse 系统成本过高 | 大流量、粗粒度阻塞与 workspace |

整页读和 slice gather 是读取策略，不是质量选择。读取额外 KV 时仍必须遵守 M。

### 9.2 用服务向量，而非一个 bytes 分数

profile 输入：source tier、bytes、span 数、页内密度、GQA mask 密度、group size、dtype、CPU workers、GPU batch/tiles、queue depth。动作成本是：

\[
v(a)=(B_{CXL},B_{DRAM},B_{H2D},B_{D2H},t_{CPU},t_{GPU},M_{workspace},n_{launch}).
\]

它们不能直接相加。对当前依赖和资源预约，选择预计 layer 最后完成时间最低的动作，满足 mask 不变、epoch、容量 credits 和 bounded submission。性能模型从微基准查表/分段拟合开始，不先训练复杂 predictor。

```text
for each finalized KV group:
    union + head masks; resolve snapshot locations
    schedule selected resident KV on GPU
    form cold runs by tier/epoch; preserve GQA sharing
    enumerate CPU-partial and GPU-pack for coarse run bundles
    estimate finish time including queues and final merge
    greedily assign large bundles, then coalesce small ones
    split only if benefit exceeds plan + sync + merge overhead
    issue with buffer credits and bounded outstanding bytes
```

MVP 只保留 resident、cold bulk、cold residual 少量分区，并设置最小任务大小和最小收益。不是逐 token 调度，不在线解大型整数规划。全 CPU、全 GPU、whole-group 都是合法候选，复杂计划不必被采用。将同一物理 token 的多个 head 贡献随同一 bundle 分配，避免 CPU/GPU 双重取数；若允许例外，必须明确计入复制成本。

### 9.3 Break-even 必须把整个路径算进去

CPU 路径含 Q 可用、源数据读、CPU attention、state copy、join；pack 路径含 gather/pack、H2D、GPU attention、join。它们各自有重叠，不能用串行求和或无条件 max 代替 timeline。selection 已收到 Q 就不再重复计 Q 成本；一次共享读也不能重复计费。

重排另满足摊销条件：预计未来有效使用次数×每次净节省，超过构建、重排、映射和维护的总成本。次数只能预测，不用测试期后见信息。第一版不依赖重排才成立。

## 10. Kernel 方案：复用什么，改什么

### GPU

复用 FlashInfer/成熟 paged attention 的 online softmax、split-KV 与归约。[R2] group-shared 模式先复用现成 backend；head-specific 模式先有独立逻辑行的正确基线，再评估有限的 masked GQA adapter。

拟议 CTA 绑定 `(request, layer, KV group, KV chunk)`，共享加载 K/V tile，按 head membership 计算，每 head 持有 FP32 统计量；padding 不进 softmax。短选集、r 小时 SIMT 可能胜过 Tensor Core，不能预设矩阵指令总是更快。tile 由 SMEM、寄存器、occupancy 和 shape profile 决定。

一个 packet 含多个 descriptors，批量 kernel 消费它们，避免每 page 一个 launch。只有 profiling 表明 mask 浪费显著，才试 **head-membership signature 分桶**：同一 mask 的 tokens 组成 tile。r heads 最多有 2^r−1 个非空桶，排序、短桶和 workspace 可能更贵，先限制高频桶+一般路径。这是可选消融，不是另一项必做系统。

### CPU

以 group span/bundle 为任务，NUMA 绑核；SIMD 计算 qK、mask、softmax 统计和加权 V。ANN 已算出的最终真实 logits 可以在语义一致且仍可用时复用；近似距离、压缩分数、变换后的分数不能无条件当作 attention logits。缓存和同步也计成本。

### 接口与图执行

post-RoPE K 重排保持原语义，不能按 compact 新下标重做 RoPE。ALiBi、soft cap、scale 和原始 position 由 adapter 明确处理。CPU 外部依赖难以塞进单个全模型 CUDA Graph；MVP 在 attention 边界切图，固定 workspace、有限 shape buckets、events。计入 plan 更新、host wait 和 graph/launch。各层选集动态变化，不假设一个 plan 可跨全部层复用。

## 11. 数值归并、full 路径与“不适合 ANN 的层”

内部状态建议为 FP32 `(m,z,u)`：m 是局部最大 logit，z=Σexp(s−m)，u=Σexp(s−m)v。对不相交分区 A、B：

\[
m=\max(m_A,m_B),\quad z=e^{m_A-m}z_A+e^{m_B-m}z_B,
\]
\[
u=e^{m_A-m}u_A+e^{m_B-m}u_B,\qquad o=u/z.
\]

空分区带 validity，避免 −∞−(−∞) 产生 NaN。backend 返回 `(o,LSE)` 时，通过明确 adapter 转换，核对对数底、scale、dtype，不能仅按 shape 合并。这是已有 online-softmax 机制，不是新贡献。[R2,R14]

| 诊断 | 决策 |
|---|---|
| exact top-k 也不合格 | 加预算、尾部估计或 full；不等于所有稀疏算法都失败 |
| exact top-k 合格而 ANN 不合格 | OOD/index recall 问题；改检索或探测 |
| 质量合格但执行慢 | 物理规划选扫描/其他路径；不是 ANN 质量问题 |

full 模式历史地址提前已知，可顺序预取。若从 sparse 扩为 full，必须只算已算集合的补集，或废弃旧 partial 后全量重算，择一并计费。不能重复 merge。尾部采样/估计另需带权 numerator/denominator 契约，MVP 不把估计量当精确 partition。

MVP sparse/full 由独立离线校准或已有可靠 selector 决定。在线候选分数无法证明未检索部分无关，不能画一个无成本的“质量失败自动 fallback”。ANN recall、保留 softmax mass、attention/logit error、自由生成任务质量分别报告。

## 12. CXL bandwidth：研究路径何时切换

v2.1 将导航溢出、候选 K 精排、最终 V 获取、K 重读、full streaming 与维护分开计量。第7.6节的候选 K 流量发生在 finalize 之前；链路仲裁和 credits 必须贯穿检索与 attention 两阶段，不能仅优化 selection 之后。

容量固定时改变真实有效带宽/竞争，观察最佳路径边界：CXL 是共同瓶颈时，CPU partial 与 GPU pack 都要读数据，换计算设备可能无用；H2D/物化瓶颈时 CPU 回小状态可能占优；CPU 饱和时 GPU 可能占优；mapped access 可用时 pack 可能多余，也可能减少 SM 远端等待。

记录 CPU NUMA、root complex、PCIe generation/lanes、CXL device 数量、read/write 并发、mapping/coherence 和 CPU cache 命中。软件限速只作服务速率敏感性实验，不能复现全部 CXL 行为。硬件 counters 不可用时区分 requested bytes、copy bytes、模型估计，不伪装成物理实测。

共享链路采用有界 chunk/queue depth、必要读优先、预取用剩余 credits、维护保底。已提交 DMA 不能任意抢占。CPU attention 与 GPU gather 共读 CXL，不能各按独占满速估算。

为了隔离索引与 KV 的作用，交叉测试：索引在 DRAM / 热导航 DRAM+冷部分 CXL；KV 在 DRAM / CXL。先 equal-capacity/equal-batch，再测容量约束下最大可行并发；大 DRAM baseline 保留作成本/容量对照。

## 13. Pipeline parallelism 的可实现版本

先单 GPU，后两 GPU 连续 layer partition。stage 持有本层权重、索引逻辑空间和 KV metadata；CXL 可共享，但队列按实际链路/设备管理。stage 之间传 activation，不在每 token 迁移整段历史 KV。

区分 `address-ready` 和 `compute-ready`：full 层历史地址在 Q 前已知，可有限预取；sparse 地址通常在真实 Q+selection 后确定。数据先到，不代表下一层 attention 可先算。

可重叠的是 stage 0 的请求 B 与 stage 1 的请求 A、当前层不同就绪 group、CPU residual 与其他独立 GPU 任务。单请求跨层和下一 token 的因果链不能被队列消除。PP 要比较均分层数、隔离 layer-time 平衡、共置路径成本平衡；计入 selection、copy、merge、MLP、activation、microbatch GEMM 效率和 CXL 排队。

**值得测试的反直觉方向：**将 sparse/full 层平均分给两个 stage 未必更快。两 stage 同时 full streaming 可能争抢同一 CXL 链路；略不均衡的分区，或错开已知地址预取，可能减少稳态停顿。需要真实双 GPU timeline 验证，不用理想 max(stage time) 模型证明。

## 14. 推荐的论文主张与可选方向

建议叙事为 **同一逻辑 attention mask 的物理执行计划选择**。核心对象是 `(真实 mask，物理位置，驻留，ready time)`：ANN/OOD 决定检索工作和选集形态，CXL 改变路径价格，GQA 决定共享边界，PagedAttention 提供映射，kernel 消费执行计划。

三个候选贡献按强弱排列：

1. **post-selection footprint**：从真实选集得到轻量物理统计，超越预算上界。代价是更晚规划，必须证明收益大于延后与统计成本。
2. **intra-group mixed execution**：同一 group 的 bulk 与 residual 走不同设备，保持逐 head mask、共享读取和正确归并。
3. **bounded materialization**：active descriptors、有界 packet、resident 原地与状态归并统一实现。第三项本身先例很强，是支撑而不是独立 novelty。

备选 kernel 方向是 membership-aware packing，对照 full-group masked、per-head、有限 signature buckets。若现成 kernel 已很高效，就停止。第一篇不要同时新建 OOD 图、学习布局、误差证书和多 GPU 调度。

## 15. 实验：必须能推翻主张

### E0 正确性

固定真实 Q/K/V 与 S_h，对相同 mask 的 reference 检查输出/统计量。覆盖重叠/不重叠 heads、空分区、尾页、position、极端 logits、resident/cold overlap、append epoch、full 补集和失败重试去重。文本“看起来合理”不足以证明实现正确。

本轮已做参考协议检查：NumPy/FP64 下1,560组 masked partition merge，覆盖不同长度、head数量、空分区和较大 logits，最大绝对差异2.442×10⁻¹⁵；另核对架构实例的分区不重不漏及字节算术。这只验证分解/归并的参考数学实现，不验证 GPU/CPU kernel、epoch 并发实现、模型质量或性能。

### E1 固定选集的路径相图——最优先

两个模型、多层、多任务采集真实 decode trace；合成数据仅补角落。扫描 unique bytes、span 数、页内密度、GQA overlap、HBM hit、CPU 核数、batch、CXL 竞争。对同一 S_h 比较 CPU partial、whole-group GPU、pack-direct、mapped（可用时）、mixed。计入 selection、plan、gather、传输、kernel、merge 和同步。

关键问题：真实 group 是否分布在不同路径优势区？同一 group 内是否存在值得拆分的异质性？若固定/whole-group 路径总接近最优，停止动态 planner。

### E2 固定质量的索引/布局

检索主矩阵采用第7节五种配置：IVF-Flat、IVF-SQ8、原精度 Roar、Roar-SQ8、Roar-TQ5。原精度 K 的 DRAM/CXL 放置、K lease 与 batch traversal 单独消融。压缩 IVF 与压缩 Roar 使用共同重排/attention 后端；先同质量与同资源比较，再测容量边界。

至少包含 SIMD scan/page summary、已有 OOD ANN、可运行的 IceCache/RetroInfer 类 selector。比较 full、exact top-k、ANN，拆开截断与索引误差。位置页与语义页固定质量或读取预算，不固定 top-k page 数。记录索引容量、节点/边/K 流量、构建、delta、维护和重排。

OOD 应操作化为 prefill 与 decode query 分布差异、multi-turn 主题切换和长生成漂移。检索目标使用模型实际 scaled inner product 与正确 RoPE 表示，不能为方便把它换成 cosine。索引校准与测试请求分离，不使用未来 decode Q 建索引。除了最终 recall，记录为达到相同质量访问了多少节点、候选 K 和 cache lines。

新前缀与复用前缀分开。固定前缀离线建索引不能从新前缀 TTFT 中消失。[R5] 若索引主导，就先解决索引或换简单 selector，而不是让 planner 掩盖瓶颈。

### E3 CXL 因子与 E4 服务/PP

执行第12节交叉设计，再加真实竞争。服务包含混合长度、multi-turn 主题变化、长输出、突发与新旧前缀。质量包括长文 QA、摘要、RULER 多针/变量/聚合计数与自由生成；teacher-forced trace 只用于隔离诊断。

报告 TTFT、逐 token ITL p99、逐请求平均 TPOT p99、tokens/s、SLO goodput、全到达请求达标率、拒绝/超时、CPU 占用、峰值三层容量。索引构建、workspace、预取浪费、维护全部计入。PP 增加 stage timeline、activation bytes、CXL queue 和 bubble。

### 强基线与消融

| 基线/消融 | 要排除的虚假收益 |
|---|---|
| 同 selector + 优化 whole-group CPU/GPU 分配 | 不是只赢串行实现 |
| SPIN 类高效 gather/cache + packet 直接 attention | 不是只省掉低效 memcpy/scatter |
| RetrievalAttention 固定 CPU history + GPU hot | 不是首次 partial 合作 |
| 最佳固定路径、静态查表、高需求层静态驻留 | 动态机制是否必要 |
| 冻结 mask 的 per-head/group-masked/signature | 分离语义变化与 kernel 改善 |
| 无 group 内切分、无 packet 合并、无竞争反馈 | 各机制能否覆盖成本 |
| 测量后离线选最佳路径的 oracle | 估计上界，不冒充在线系统 |

SAC/ECHO/native sparse 模型另设 track，不与 dense 后稀疏化直接比一个吞吐值。未运行原系统时标注“机制复现”。预算、模型精度、CPU cores、GPU/DRAM/CXL 容量和质量要求保持公平；同时报告 equal-batch 与最大 SLO 可行 batch。

## 16. MVP 与停止条件

**MVP-A：正确可测。** 单 GPU、一个 GQA 模型、IP IVF-Flat；继而 IVF-SQ8 + 原 K 精排/K lease；GPU hot+CPU historical partial；DRAM/CXL 可替换后备；固定 epoch/delta；批量 state 回传，全链路 trace。

**MVP-B：强简单对照。** whole-group GPU-pack、pack-direct、静态 profile 路由；head mask 不兼容时使用明确正确的基线。测动态 oracle 剩余空间。

**MVP-C：一个主机制。** post-selection footprint + 有限 group 内分流，控制 packet/join/plan 成本；再扩两模型、服务与真实竞争。PP 在单 GPU 净收益成立后做。

先产出三张研究图：①同选集 CPU/GPU 路径相图；②名义 sparsity→unique bytes→真实流量→时间分解；③whole-group 与细分流的净收益及 planner/merge 开销。

停止条件：固定/whole-group 已接近 oracle；主要瓶颈在索引、MLP/权重等且 Amdahl 空间不足；mixed join/mask/sort 吃掉收益；收益依赖改 mask、降质量或额外资源；CXL 没有容量/成本价值。可设约15–20%稳定端到端净空间作为内部投入门槛，但不是投稿标准。

## 17. 来源与核验范围

本次向量模块重点重新核对以下一手材料：

- **[U2-C]** 用户提供 `01-SIGMOD27_CXL_Vector_P1297.pdf`，重点§3–5：DRAM graph/codes、CXL 原向量、SQ8/TQ5、批量 distance、prefetch、visited。组合 RoarGraph、KV 追加与 K lease 是本文提出的适配，不是稿件已验证结果。
- **[R15] RoarGraph**，[论文全文](https://arxiv.org/html/2408.08933v1)，§4、§5.7、§6；[官方实现](https://github.com/matchyc/RoarGraph)。核对投影流程、IP、建图样本和 offline insertion。
- **[R16] Faiss metric**，[官方说明](https://github.com/facebookresearch/faiss/wiki/MetricType-and-distances)：IP 与 cosine 的区别。
- **[R17] Faiss indexes**，[官方索引目录](https://github.com/facebookresearch/faiss/wiki/Faiss-indexes)：IVF-Flat、IVF-SQ、nprobe、IDs/encoding。通用索引不自带本文的分层 serving 协议。
- **[R18] RetrievalAttention 官方仓库**，[代码入口](https://github.com/microsoft/RetrievalAttention)，2026-09-29 所见 main README 为 RetroInfer；复现实验需锁定对应版本。


- **[U1]** 用户 HTML《KV Cache：从 Lecture 10 到 Kernel Efficiency》，2026-09-22，全文阅读，尤其§3–8。
- **[U2]** 原八份用户来源：CXL-Vector 匿名稿、RetrievalAttention v3、RoarGraph、Quest、ECHO、ScoutAttention、PagedAttention DOI、KV survey。沿用已核验结论；匿名稿平台数字不代替 GPU+CXL 实测。
- **[R1] IceCache**，[全文](https://arxiv.org/html/2604.10539v1)，§4；[官方实现](https://github.com/yuzhenmao/IceCache)。
- **[R2] FlashInfer**，[全文](https://arxiv.org/html/2501.01005v1)，§2–3、App. A/B/D。GPU-resident 稀疏 gather 结论不能直接外推 CXL。
- **[R3] SPIN**，[全文](https://arxiv.org/html/2604.26837v1)，§4–5，预印本。
- **[R4] Fluxion**，[全文](https://arxiv.org/html/2605.07719v1)，§5.3–6.2，预印本。
- **[R5] RetrievalAttention**，[NeurIPS 2025 正式版 PDF](https://proceedings.neurips.cc/paper_files/paper/2025/file/4e36d4049fb0fea195a8267c8dcd0824-Paper-Conference.pdf)，§4/Table4/App.A/C。用户 v3 与正式版性能数字不混用。
- **[R6] Strata**，[OSDI 2026 官方页面](https://www.usenix.org/conference/osdi26/presentation/xie-zhiqiang)，[PDF](https://www.usenix.org/system/files/osdi26-xie-zhiqiang.pdf)，§4。
- **[R7] DirectKV**，[OSDI 2026 官方页面](https://www.usenix.org/conference/osdi26/presentation/luo)，[PDF](https://www.usenix.org/system/files/osdi26-luo.pdf)，§2/4/5。
- **[R8] Swarm**，[全文](https://arxiv.org/html/2603.17803v1)，§5–6，预印本。
- **[R9] CompactAttention**，[全文](https://arxiv.org/html/2605.16839v1)，§3/App.B；[官方实现](https://github.com/jiwonsong-dev/CompactAttention)，预印本。
- **[R10] SAC**，[全文](https://arxiv.org/html/2606.19746v1)，预印本。
- **[R11] Beluga**，[全文](https://arxiv.org/html/2511.20172v1)，§4–5。
- **[R12] FlashInfer sparse API**，[官方文档](https://docs.flashinfer.ai/api/sparse.html)，本次查询0.7.0，重点 GQA mask 契约。
- **[R13] TensorRT-LLM Sparse Attention Development Guide**，[官方文档](https://nvidia.github.io/TensorRT-LLM/developer-guide/sparse-attention-development-guide.html)，index layout contract。
- **[R14] FlashInfer state merge**，[官方接口](https://docs.flashinfer.ai/generated/flashinfer.cascade.merge_state.html)。adapter须验证实际状态格式。

保留对照：[RetroInfer](https://www.vldb.org/pvldb/vol19/p1016-lu.pdf)、[Quest](https://arxiv.org/abs/2406.10774)、[PagedAttention](https://arxiv.org/abs/2309.06180)、[ECHO](https://www.usenix.org/conference/osdi26/presentation/liu-guangda)、[ScoutAttention](https://arxiv.org/abs/2603.27138)、[LayerKV](https://arxiv.org/abs/2410.00428)、[Verified Sparse Attention](https://arxiv.org/abs/2510.05688)、[Louver](https://arxiv.org/abs/2605.06763)。质量证书和尾部估计作为策略对照，不在 MVP 同时重做。

本轮检索核验日期2026-09-29。以上形成的是更具体、可实施、可证伪的研究假设，尚未确认新颖性或性能优势；投稿主张应由实验决定。
