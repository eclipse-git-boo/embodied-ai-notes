---
layout: page
title: Knowledge Insulating VLA：π₀.₅的知识隔离机制
---

原文：[官方 PDF](https://www.physicalintelligence.company/download/pi05_KI.pdf)；本地 PDF：[`VLA/论文/pi05-knowledge-insulating-vla.pdf`](../../../VLA/论文/pi05-knowledge-insulating-vla.pdf)。这是 π₀.₅路线的重要补充论文，应与主论文连读。

## 它补上了什么

标准 VLA 面临三角矛盾：自回归离散动作有利于利用 VLM 表征但推理慢；flow 动作快但新初始化 action expert 的梯度会破坏预训练主干；只冻结主干又会让机器人表征不足。论文的组合回答是：

1. 同时训练离散 FAST 动作/语言 next-token 目标和连续 flow-matching 动作目标；
2. 把 VLM 图文与机器人规划数据共同训练，降低机器人适配时的知识遗忘；
3. 阻断 action expert 回传到预训练 backbone 的梯度（knowledge insulation），同时让主干直接接受离散动作监督。

## 对 π₀.₅微调的启示

这说明“只调 LoRA/只调 action head”不是普适配方。当前 openpi 的公开 PyTorch 路径有功能边界（例如 π₀-FAST、LoRA、mixed precision 等支持状态随版本变化），实际复现必须以锁定 commit 的官方 README 和配置为准。先复现官方 LIBERO 配置，再复制/修改官方数据映射，不能凭论文公式自行拼装训练图。
