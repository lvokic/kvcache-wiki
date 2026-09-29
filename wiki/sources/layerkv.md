# LayerKV: Optimizing Large Language Model Serving with Layer-wise KV Cache Management

**作者：** Yi Xiong, Hao Wu, Changxu Shao, Ziqing Wang, Rui Zhang, Yuhong Guo, Junping Zhao, Ke Zhang, Zhenxuan Pan。**版本：** arXiv:2410.00428 v3，2024-10-09。
**本地原件：** [layerkv-arxiv24.pdf](../../raw/papers/layerkv-arxiv24.pdf)
**来源：** [arXiv:2410.00428](https://arxiv.org/abs/2410.00428)

## 摘要与设计

LayerKV 将 GPU KV blocks 的分配、管理和卸载按 transformer layer 细分，并配合 SLO-aware scheduler 降低长上下文请求造成的排队延迟。作者在 7B–70B 模型和多种 GPU 配置上报告 TTFT 与 SLO 违约改善。

## 与本研究的关系

- 它是按层做 KV residency/offload 和容量调度的相关 baseline；层级预算、layer-aware placement 和调度兼容性并非空白。
- LayerKV 的动机是容量冲突与 TTFT queueing，不是 sparse selector 返回后的 token-level physical span，也不使用 ANNS/CXL/CPU partial attention。
- 用它对比时应控制 admission、层预算与请求负载；否则把 TTFT queueing 改善和 decode 每步物理读取收益混为一谈。

## 相关页面

- 参见 [SPIN](spin.md)、[HiSparse](hisparse.md) 和 [KV cache 管理总览](../topics/kv-cache-management.md)。
