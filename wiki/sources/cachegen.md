# CacheGen: KV Cache Compression and Streaming for Fast Large Language Model Serving

**作者：** Yuhan Liu, Hanchen Li, Yihua Cheng, Siddhant Ray, Yuyang Huang, Qizheng Zhang, Kuntai Du, Jiayi Yao, Shan Lu, Ganesh Ananthanarayanan, Michael Maire, Henry Hoffmann, Ari Holtzman, Junchen Jiang  
**发表：** ACM SIGCOMM 2024  
**本地 PDF：** [raw/papers/cachegen-sigcomm24.pdf](../../raw/papers/cachegen-sigcomm24.pdf)，arXiv v6（2024-07-19）  
**出处：** [ACM DOI 10.1145/3651890.3672274](https://doi.org/10.1145/3651890.3672274) · [arXiv:2310.07240](https://arxiv.org/abs/2310.07240) · [作者提供的会议版 PDF](https://cs.stanford.edu/~keithw/sigcomm2024/sigcomm24-final1571-acmpaginated.pdf)

## 摘要

CacheGen 面向复用长 context 时的网络加载延迟。它利用 KV tensor 的分布特性编码更紧凑的 bitstream，并按带宽变化调整 KV 不同部分的压缩级别；带宽不足时也可以选择让模型重算部分 KV。目标是在减少传输量的同时控制生成质量。见原文 §4、§7。

## 关键证据

论文报告 KV 大小减少 3.5–4.3×，context fetch 和 processing 的总延迟减少 3.2–3.7×，且生成质量影响很小。这些数字来自论文使用的模型、数据集和传输设置；和 [Mooncake](mooncake.md) 的整体 serving capacity 结果衡量的不是同一指标。出处：[SIGCOMM 论文 PDF](https://cs.stanford.edu/~keithw/sigcomm2024/sigcomm24-final1571-acmpaginated.pdf) · [arXiv 摘要](https://arxiv.org/abs/2310.07240)。

## 系统边界

CacheGen 重点优化 KV 的编码和传输数据路径，不定义一个长期共享 cache 的全局命名、ownership 或淘汰服务。因此它更像可与 P/D disaggregation 或远程 cache 组合的传输机制；这是基于本文范围的综合判断，尚不是跨系统组合的实验证据。

