# QUEST: Query-Aware Sparsity for Efficient Long-Context LLM Inference

**作者：** Jiaming Tang, Yilong Zhao, Kan Zhu, Guangxuan Xiao, Baris Kasikci, Song Han  
**发表：** ICML 2024，PMLR 235，47901–47911  
**原始文件：** [raw/papers/quest.pdf](../../raw/papers/quest.pdf)  
**正式论文：** [PMLR 论文页与 PDF](https://proceedings.mlr.press/v235/tang24l.html)

## 摘要

Quest 是 query-aware 的 KV page 选择方法。它在每个 page 上保存 key 各维度的最小值和最大值，用当前 query 给 page 计算一个注意力重要性上界，再只加载分数最高的 pages 执行 attention。它将稀疏模式的决定推迟到 decode 时，因此同一份上下文可随当前 query 选出不同的历史信息。见论文 §3、Fig. 3。

## 主要结果

- 在 Llama2-7B、32K 序列、2048 token budget 的测试中，论文报告 self-attention 比 FlashInfer full-cache 基线快 7.03×；端到端 decode latency 最多降低 2.23×。前者是 attention 子路径，后者是整体 decode 指标，不能互换。见 §4.3、Fig. 9–10。
- 在 LongBench 任务上，作者报告可在较少保留 token 的情况下达到与 full cache 相近的准确率；不同任务需要的 token budget 不同。见 §4.2–4.3。

## 边界与限制

- 这是 GPU KV page 筛选算法：page min/max 元数据服务于 query-aware pruning。它不是通用 ANNS 图索引，也没有研究把完整 KV 或索引放入 host/CXL memory pool。
- 速度和准确率依赖 page size、token budget、模型及任务；“近似无损”是论文评估结果，不是对任意模型和 workload 的精确保证。
- 7.03× 是特定序列长度与预算下的 attention kernel 结果，系统容量、远端内存访问及多租户 serving 不是论文重点。

## 与本 wiki 的连接

- 与 [DeepSeek-V3.2](deepseek-v3-2.md) 对照：Quest 用 page 元数据估计 query 相关性；DSA 用训练出的 lightweight indexer 为 token 打分。
- 与 [RetrievalAttention](retrievalattention.md) 对照：Quest 以 page 上界选择候选；RetrievalAttention 以向量检索索引在 CPU 上定位候选 KV。
- 与 [sparse attention、CXL 与 ANNS 综合页](../topics/sparse-attention-cxl-anns.md) 一起读。两篇工作分别涉及选择器和外置 ANN 索引，但 Quest 本身没有 CXL/ANNS 路径。
