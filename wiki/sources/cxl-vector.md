# CXL-Vector: High-Throughput Graph-Based Vector Search on CXL Memory

**作者/版本：** 匿名 SIGMOD ’27 投稿稿，编号 P1297；当前 PDF 是用户提供的工作稿，录用状态未确认，以下性能均按稿件报告处理。
**本地原件：** [SIGMOD27_CXL_Vector_P1297.pdf](../../raw/papers/SIGMOD27_CXL_Vector_P1297.pdf)

## 摘要

CXL-Vector 面向 commodity memory-only CXL 上的图近似最近邻搜索。它将图拓扑和紧凑向量码放在本地 DRAM，将原精度向量留在 CXL，只对最终候选访问原向量重排。它是本 wiki 中与本 proposal 最直接相关的前序工作：已经覆盖“DRAM 导航、CXL 原向量、候选精排”这条分层 ANN 路径。

## 设计与稿件报告

- 支持 HNSW 和 NSG；对 SQ8/TQ5 码使用批量图扩展、AVX-512 VNNI 点积和 mask-and-pack 候选维护。稿件 §2–4。
- 使用 chunk-ahead 预取、2 MiB huge-page 准备和 cache-resident blocked Bloom filter 降低码访问、TLB miss 和 visited 检查开销。性能取决于稿件的批处理与内存配置。
- 摘要报告在 Recall@10=0.90 配置下，吞吐相对 HNSW 高 3.9–7.2 倍；并报告相对 SymphonyQG 的吞吐和本地 DRAM 用量改善，以及相对 SSD 路线的优势。具体值需以稿件中的对应数据集和基线表为准。

## 与当前 proposal 的边界

- 稿件证明图 ANNS 的压缩导航和 CXL 原向量重排已有先例，因此不能把这种放置方式本身作为新颖性主张。
- 它评估通用 HNSW/NSG 向量搜索，不是 attention Q/K 的 OOD 检索；不测 per-head mask、GQA、attention 输出误差、KV 的 V payload 或 token 追加生命周期。将 RoarGraph、真实 attention query 和 KV 物理规划接入其中属于后续设计与实验，不是该稿件结果。
- 这是匿名投稿稿，不应描述为已发表的 SIGMOD 论文。

## 相关页面

- [CXL-ANNS](cxl-anns.md) 提供 CXL 图检索的另一种软硬件协作路径。
- [RoarGraph](roargraph.md) 与 [RetrievalAttention](retrievalattention.md) 涉及 OOD 图检索和 attention-aware 搜索。
- 参见 [Sparse Attention、CXL 与 ANNS 综合页](../topics/sparse-attention-cxl-anns.md)。
