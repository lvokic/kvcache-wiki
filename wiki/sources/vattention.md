# vAttention: Dynamic Memory Management for Serving LLMs without PagedAttention

**作者：** Ramya Prabhu, Ajay Nayak, Jayashree Mohan, Ramachandran Ramjee, Ashish Panwar  
**发表：** ASPLOS 2025, pp. 1133–1150  
**本地 PDF：** [raw/papers/vattention-asplos25.pdf](../../raw/papers/vattention-asplos25.pdf)，作者提供的正式版  
**出处：** [ACM DOI 10.1145/3669940.3707256](https://doi.org/10.1145/3669940.3707256) · [arXiv:2405.04437 v3](https://arxiv.org/abs/2405.04437) · [作者 PDF](https://apanwariisc.github.io/publications/asplos-2025-vattention/vattention-asplos25.pdf)

## 摘要

vAttention 用 CUDA virtual memory management APIs 分离 GPU 虚拟地址空间和物理 backing 的分配时机：KV cache 在虚拟地址上保持连续，同时物理内存按需映射，以减少碎片。作者针对 CUDA VMM 的限制加入 LLM 特定优化，目标是让既有 attention kernels 无需显式支持 paged layout。见原文 §3–5。

## 和 PagedAttention 的设计取舍

- [PagedAttention](pagedattention.md) 在应用层维护逻辑到物理 block 的映射，kernel 知道并遍历 blocks；vAttention 把动态物理 backing 放到 VMM 层，保留 kernel 所见的虚拟连续性。
- 论文报告其吞吐最高比基于 PagedAttention 的 FlashAttention/FlashInfer kernel 高 1.23×。这是作者评测条件下的结果，不表示任意 GPU、驱动、kernel 或 serving stack 都有同样收益。出处：[ACM 论文页](https://doi.org/10.1145/3669940.3707256)。
- 设计比较的核心是 kernel/API 复杂度、VMM 支持和性能，而非 cache 的远程 ownership 或跨会话保留策略。

## 边界

vAttention 和分页 block manager 都在解决动态分配与碎片问题，但抽象层不同。适用性取决于虚拟内存 API、页映射粒度和系统栈集成；不能仅凭“连续虚拟地址”推断底层物理页连续。

