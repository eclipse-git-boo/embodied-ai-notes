---
layout: page
title: "IRASim：动作条件机器人视频模拟器"
description: "以扩散 transformer 学动作条件视频动力学，服务于预测和控制。"
---

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/irasim-02.png' | relative_url }}" alt="IRASim 论文图示"><figcaption>论文图示摘录（PDF 第 2 页）。来源：[IRASim](https://arxiv.org/abs/2406.14540)。</figcaption></figure>

## 1. 快读卡片

| 项目 | 内容 |
|---|---|
| 论文 | *IRASim: Learning Interactive Real-World Action Simulators*（2024） |
| 原文 / 代码 | [arXiv:2406.14540](https://arxiv.org/abs/2406.14540) · [官方仓库](https://github.com/bytedance/IRASim) |
| 核心 | 从真实交互数据学习 action-conditioned video simulator，用视频生成来模拟动作后果。 |
| 学习价值 | 是从“视频模型”走向“可用于机器人交互的世界模型”的中间桥梁。 |

## 2. 动机与相关工作

视觉预测模型若忽略动作，无法用于控制；物理模拟器若只在仿真中拟合，又受 sim-to-real 限制。IRASim 直接在真实机器人交互视频上训练，目标是当给出动作时，预测包含机器人和物体变化的未来画面。它与 UniPi 都使用视频预测，但 IRASim 更强调交互式 action simulator 的学习。

## 3. 方法

论文使用扩散 Transformer 处理视频 latent/patch，并将动作作为时序条件注入。训练以真实轨迹的视频与动作对为样本，生成端给定历史帧和未来动作序列生成未来片段。若用于控制，可在候选动作之间比较生成结果，但基础模型本身并不自动提供任务奖励或安全证明。

<div class="method-flow"><span>历史视频 + action sequence</span><b>→</b><span>动作条件 DiT</span><b>→</b><span>未来交互视频</span><b>→</b><span>规划打分 / 数据增强</span></div>

## 4. 实验与讨论

实验关注生成质量、动作可控性以及下游效果。阅读时区分三件事：像素/视频指标、动作敏感性、闭环控制收益。一个模型即使 FVD 较好，也可能对夹爪闭合时刻、接触滑移等控制关键变量不敏感。论文不应被读成“生成得像就等于物理正确”。

## 5. 可复现性审计

| 项目 | 检查点 |
|---|---|
| 数据 | 严格对齐图像、动作和控制频率；每次插值都应记录。 |
| 训练 | 先用短 horizon 和固定相机检查 action-conditioning 是否有效。 |
| 评估 | 做 action swap / counterfactual：同一图像换动作后预测必须随之变化。 |
| 局限 | 长时程误差累积、遮挡、罕见接触和相机位姿变化都会损害预测。 |

## 6. 算力、硬件与风险（学习规划，不在本机部署）

| 目标 | 建议配置 | 说明 |
|---|---|---|
| 小视频预测实验 | 24–48 GB GPU、NVMe 视频缓存 | 先在离线数据上做 action swap。 |
| 视频模型后训练 | 48–80 GB GPU / 多卡 | 分辨率、帧数和 diffusion steps 决定成本。 |
| 全量预训练 | 多卡集群与大量交互视频 | 不作为个人复现目标。 |
| 真机规划 | 保守短 horizon、碰撞/速度安全层 | 不能把生成视频当作碰撞检测器。 |
