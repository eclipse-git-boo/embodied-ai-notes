---
layout: page
title: IRASim：动作条件机器人视频世界模型
---

原文：[arXiv 2406.14540](https://arxiv.org/abs/2406.14540)；代码/项目：[gen-irasim.github.io](https://gen-irasim.github.io/)；本地 PDF：[`WAM/论文/IRASim-2406.14540.pdf`](../../../WAM/论文/IRASim-2406.14540.pdf)。

## 问题

机器人 world model 不只要生成“像真的”视频，还要精确遵循每一帧对应的动作。IRASim 输入历史观测和动作轨迹，用 diffusion transformer 生成轨迹对应视频，并在每个 Transformer block 加入逐帧动作条件以强化 action-frame alignment。

## 它为什么是本项目的 WAM 首选阅读

- 直接面对机械臂动作 chunk，而不是抽象游戏环境；
- 展示两种实际用法：在模型里评估策略、从候选轨迹中规划；
- 也暴露了最重要风险：模型可能在视频里“幻觉式成功”，优化器会利用这个漏洞（world-model hacking）。

未来若 π₀.₅基础策略已稳定，最小 WAM 扩展应先做“从 K 个 π₀.₅候选动作中用预测评分重排”，并与不重排的盲测成功率和碰撞率比较；不要先训练大视频生成器。

