# WAM：World Action Model

先阅读 [WAM 核心概念](01-核心概念.md)，再依照 [六周学习计划](../学习笔记/04-六周学习计划.md) 进入论文；本目录当前仅供学习，不在本机执行代码。

## 学习主线

1. [DreamerV3](https://arxiv.org/abs/2301.04104)：世界模型、想象 rollout 与 actor-critic 的基础。
2. [Genie: Generative Interactive Environments](https://arxiv.org/abs/2402.15391)：学习 latent action 与交互式视频世界。
3. [World Action Models: The Next Frontier in Embodied AI](https://arxiv.org/abs/2605.12090)：用作概念与论文地图；阅读时区分综述观点和可复现实证。

## 最小可做实验（SO-ARM101 数据）

目标不是训练一个通用视频生成器，而是检验预测是否能辅助控制。

- 输入：当前及过去 2–4 帧、机器人 proprioception、候选动作 chunk。
- 目标：预测 0.25–1 秒后的末帧，或预测任务相关状态（物块/夹爪/盒子的相对位置）。
- 模型：小型 CNN/ViT 编码器 + action-conditioned temporal transformer，先以 L1/latent loss 训练。
- 用法：由 ACT/VLA 产生 K 个候选动作，WAM 对未来碰撞、越界、离目标距离打分，选择最优候选。
- 对照：同一初始状态下，不重排的策略 vs WAM 重排；报告成功率、碰撞率、推理延迟和预测误差。

只有当离线预测与真机重排均有正向证据时，再扩大到视频生成或规划。
