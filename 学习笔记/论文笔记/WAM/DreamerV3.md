---
layout: page
title: "DreamerV3：在潜空间中想象并学习控制"
description: "通用 world-model 强化学习基线：世界模型、actor-critic 与 imagined rollouts。"
---

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/dreamerv3-02.png' | relative_url }}" alt="DreamerV3 论文图示"><figcaption>论文图示摘录（PDF 第 2 页）。来源：[DreamerV3](https://arxiv.org/abs/2301.04104)。</figcaption></figure>

## 1. 快读卡片

| 项目 | 内容 |
|---|---|
| 论文 / 代码 | [DreamerV3](https://arxiv.org/abs/2301.04104) · [官方实现](https://github.com/danijar/dreamerv3) |
| 核心 | 学习紧凑的 recurrent state-space model（RSSM），在 latent imagination 中训练 actor 与 critic。 |
| 价值 | 展示“先预测世界，再在预测世界中学习”的可扩展 RL 配方；并不直接等于 VLA。 |

## 2. 动机与相关工作

model-free RL 从真实环境反复试错，样本成本高；模型式方法若直接预测像素又难以稳定。Dreamer 系列将观测压缩到 latent state，以潜空间动力学预测未来，令策略在 imagined trajectories 上更新。DreamerV3 的重点是单一配置跨 Atari、控制、Minecraft 等领域稳定工作，而非为机器人语言指令专门设计。

## 3. 方法

世界模型包括 encoder、确定性循环状态、随机 latent、decoder、reward/continue heads。给定 \(s_t,a_t\) 预测 \(s_{t+1}\)，从真实 replay 学表征与动力学；actor/critic 再从 latent rollout 采样，最大化预期回报。实现上，KL balancing、symlog 两侧变换、百分位回报归一化和统一超参是稳定性的关键。

<div class="method-flow"><span>真实观测</span><b>→</b><span>RSSM latent world model</span><b>→</b><span>imagined rollout</span><b>→</b><span>actor / critic 更新</span><b>→</b><span>环境动作</span></div>

## 4. 实验与讨论

论文展示同一算法配置在大量任务上获得竞争力，说明世界模型可作为通用学习系统。但其观测、奖励和动作接口通常已经由环境定义；真实机械臂会额外面对相机时延、标定、碰撞成本和不可重置性。因此 DreamerV3 更适合作为 WAM 的基础概念与仿真算法对照。

## 5. 可复现性审计

| 项目 | 要点 |
|---|---|
| 代码 | 官方 JAX 实现公开；先在标准环境复验，不要直接接真机。 |
| 数据 | 以在线 replay 为主；离线机器人数据需改造成安全的 offline/world-model 方案。 |
| 关键指标 | model loss 不等于 policy return；记录 imagined value、真实回报和模型 rollout 漂移。 |
| 限制 | 无法自动处理稀疏奖励、危险探索或真实接触的分布转移。 |

## 6. 算力、硬件与风险（学习规划，不在本机部署）

| 目标 | 建议配置 | 备注 |
|---|---|---|
| 控制基准复现 | 12–24 GB GPU 或高性能 CPU | 先做仿真。 |
| 图像/机器人规模 world model | 24–48 GB GPU、快速 replay SSD | 图像序列与批量想象会占显存/IO。 |
| 大规模预训练 | 多卡集群 + 长时间环境采样 | 不适合当前 π0.5 微调目标。 |
| 真机 | 永远不允许无约束探索 | RL 探索必须由仿真、shield 或人工安全层限制。 |
