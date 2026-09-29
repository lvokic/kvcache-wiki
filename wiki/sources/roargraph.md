# RoarGraph: A Projected Bipartite Graph for Efficient Cross-Modal Approximate Nearest Neighbor Search

**作者：** Meng Chen, Kai Zhang, Zhenying He, Yinan Jing, X. Sean Wang  
**发表：** PVLDB 17(11)，2024，2735–2749；DOI 10.14778/3681954.3681959  
**原始文件：** [raw/roargraph.pdf](../../raw/roargraph.pdf)  
**正式论文：** [PVLDB 论文 PDF](https://www.vldb.org/pvldb/vol17/p2735-chen.pdf)

## 摘要

RoarGraph 面向 cross-modal ANNS：query 与 base vectors 来自不同模态，分布差异使 query 对常见“query 靠近数据点、真实近邻互相接近”的图索引假设失效。作者利用 query 分布构建 bipartite query–base 关系，再把关系投影到 base-data graph，给 query 视角下相近、但向量空间相距较远的点补出可遍历路径。见 §1、§3。

## 主要发现与结果

- 分析显示，OOD query 的真实近邻在 embedding space 里往往彼此分散；传统 proximity graph 会因此产生更多搜索跳数。RoarGraph 在三种 text/image/video cross-modal 数据集上测试，报告在 recall@k ≥ 0.9 时比所比较方法快 1.84×–3.56×。见 §1、§2、§6。
- 它保留 query 分布带来的邻接关系，同时最终索引只包含 base vectors；这点对 RetrievalAttention 有直接启发：用 query–key 关系弥合 Q/K 分布差异，再把关系折叠到 key 图中。

## 边界与限制

- 论文研究的是跨模态向量检索，不是 Transformer 的 Q/K/V 或 sparse attention；它的 OOD 问题与 Q/K 分布差异相似，但不能直接等同。
- 评测覆盖三类跨模态数据集和 recall/speed trade-off；没有研究 CXL、远程 KV payload、每层 attention 输出误差或 serving SLO。

## 与本 wiki 的连接

- [RetrievalAttention](retrievalattention.md) 明确借用 RoarGraph 的投影技术，是通用 OOD graph ANNS 到注意力向量检索的实例化。
- [CXL-ANNS](cxl-anns.md) 展示如何把大规模图索引放到 CXL memory pool，并降低图遍历的远程访问代价。
- 参见 [sparse attention、CXL 与 ANNS 综合页](../topics/sparse-attention-cxl-anns.md)。
