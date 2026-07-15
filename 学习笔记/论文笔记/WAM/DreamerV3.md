---
layout: page
title: DreamerV3：潜变量世界模型基础
---

原文：[arXiv 2301.04104](https://arxiv.org/abs/2301.04104)；代码：[danijar/dreamerv3](https://github.com/danijar/dreamerv3)；本地 PDF：[`WAM/论文/DreamerV3-2301.04104.pdf`](../../../WAM/论文/DreamerV3-2301.04104.pdf)。

## 为什么先读它

DreamerV3 不是 VLA，也不是视频式 WAM；它是理解“为什么要先预测、再选择动作”的开源基线。模型把高维观测编码为随机/确定潜变量，在潜空间学习动力学、奖励和继续概率；actor 与 critic 在想象轨迹上优化，而不必每一步向真实环境采样。

## 应掌握的图

```text
观测 o_t → encoder → latent z_t ─┬→ decoder / reward / continue
动作 a_t ────────────────────────┘
                    ↓ imagination
                 actor / critic
```

## 对 π₀.₅的关系

π₀.₅直接输出动作，是行为克隆/VLA 路线；DreamerV3 为将来引入世界模型提供评价标准：预测必须改善决策或数据效率。现在不应把 DreamerV3 与 π₀.₅微调并行实现，因为它需要奖励、环境交互和稳定 RL 训练，变量远多于 π₀.₅的小数据适配。

