---
layout: page
title: FAST / π₀-FAST：高效动作 token 化
---

原文：[arXiv 2501.09747](https://arxiv.org/abs/2501.09747)；代码：[openpi](https://github.com/Physical-Intelligence/openpi)；本地 PDF：[`VLA/论文/FAST-pi0-FAST-2501.09747.pdf`](../../../VLA/论文/FAST-pi0-FAST-2501.09747.pdf)。

## 先澄清名称

π₀-FAST不是另一篇独立论文的标题；它是 FAST 论文中，以 π₀骨干结合 FAST action tokenizer 得到的自回归 VLA。因此本笔记以 FAST 论文为准。

## 方法

普通动作 token 化把每个维度、每个时刻分别分桶。对高频控制，连续时刻高度相关，模型可以靠复制上一个 token 获得低 loss，却没有学到有效控制。FAST 的链路是：

```text
连续动作 chunk → DCT（转频域）→ 量化/字节序列 → BPE 压缩 → 自回归 token
                                                    ↓
连续动作 chunk ← inverse DCT ← reshape ← BPE 解码 ← 预测 token
```

论文报告 π₀-FAST 可匹配 π₀的任务表现且训练更快，但自回归推理仍需要逐 token 生成。openpi 当前 PyTorch 实现不支持 π₀-FAST；这是选择 π₀.₅为本项目目标的现实理由之一。

## 需要带走的工程认识

- “动作表示”是第一等设计变量，而非模型末端小细节。
- 训练快不表示部署快；token 数、KV cache 和生成顺序决定时延。
- 将 SO-ARM101 动作改成不同维数并非只改一个 shape：必须重新验证 tokenizer、统计量与控制语义。

