# VLA 学习与复现顺序

配合 [六周学习计划](../学习笔记/04-六周学习计划.md) 和 [VLA 核心概念](01-核心概念.md) 阅读；本目录当前仅供学习，不在本机执行代码。

## 必读论文（按顺序）

1. [RT-1: Robotics Transformer for Real-World Control at Scale](https://arxiv.org/abs/2212.06817)：理解大规模真实机器人数据、动作分块与控制闭环。
2. [RT-2: Vision-Language-Action Models Transfer Web Knowledge to Robotic Control](https://arxiv.org/abs/2307.15818)：理解把动作表示为 token、语言/视觉迁移。
3. [OpenVLA](https://arxiv.org/abs/2406.09246)：经典可复现开源 VLA；重点读动作 tokenization、Prismatic VLM、数据配方与 LIBERO 评测。
4. [π₀: A Vision-Language-Action Flow Model for General Robot Control](https://www.physicalintelligence.company/download/pi0.pdf)：理解 flow matching 动作头与异构机器人数据。

每篇笔记至少回答：输入观测是什么？动作如何表征？训练数据来自哪里？损失是什么？控制频率/动作 chunk 是什么？在何种 benchmark 上评测？对你的 SO-ARM101 缺少什么适配？

## 复现决策

先使用 LeRobot 格式数据训练 ACT/SmolVLA，再选 OpenVLA 的官方 LIBERO 微调做论文复现。OpenVLA 的原始机器人本体、数据格式和动作空间并不等同于 SO-ARM101，未经动作/坐标/相机适配不可直接下发真机。

openpi 是第二条研究路线：官方仓库包含 π₀、π₀-FAST 和 π₀.₅，并提供预训练权重与微调示例；应先在其支持的格式和仿真/离线评测中验证，再做本体适配。
