---
layout: page
title: "UniPi：用视频扩散模型做通用机器人规划"
description: "把动作条件视频预测用于视觉规划，而不是直接端到端动作回归。"
---

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/unipi-02.png' | relative_url }}" alt="UniPi 论文图示"><figcaption>论文图示摘录（PDF 第 2 页）。来源：[UniPi](https://arxiv.org/abs/2302.00111)。</figcaption></figure>

## 1. 快读卡片

| 项目 | 内容 |
|---|---|
| 论文 / 代码 | [UniPi](https://arxiv.org/abs/2302.00111) · [官方仓库](https://github.com/wyndwarrior/UniPi) |
| 核心 | 以动作条件视频扩散模型预测“执行候选动作后会看到什么”，再据目标/语言选择动作。 |
| 与 VLA 的区别 | 规划器显式搜索或评估未来视觉；不是一次前向直接输出最终动作。 |

## 2. 动机与相关工作

长时程操作常需先推断动作后果。传统视觉规划可能依赖昂贵任务特定模型；UniPi 尝试使用可扩展的视频扩散先验，把视觉预测作为通用 planning interface。它与 model predictive control（MPC）接近：每轮预测候选短期未来、选择动作、执行后重新观测。

## 3. 方法

给定当前图像 \(o_t\)、候选动作 \(a_{t:t+H}\) 和目标条件，动作条件扩散模型产生未来图像/latent。规划器用目标相似度或价值函数排序候选序列，执行前缀后滚动重规划。论文重点是视频模型的生成能力如何转成控制信号；计划质量取决于预测保真、目标函数和采样预算三者。

<div class="method-flow"><span>当前图像 + 目标</span><b>→</b><span>候选 action sequences</span><b>→</b><span>动作条件视频预测</span><b>→</b><span>目标打分</span><b>→</b><span>执行前缀 / 重规划</span></div>

## 4. 实验与边界

UniPi 在机器人操作和视觉控制设置中显示视频先验可辅助规划。应特别关注长时程时的计算量：每次 MPC 都需多候选、多采样步，控制频率可能成为瓶颈。视频“看起来合理”也可能在几何接触、遮挡或夹爪状态上错误，因此真实执行要以短 horizon 和频繁重观测降低模型偏差。

## 5. 可复现性审计

| 项目 | 要点 |
|---|---|
| 代码 / 依赖 | 检查其视频模型、数据预处理与动作条件格式；版本较早，依赖可能需迁移。 |
| 关键参数 | planning horizon、候选数、diffusion steps、目标距离、执行前缀长度。 |
| 验证 | 分别测 video prediction、planner selection、闭环 success；不要只看生成视频。 |
| 局限 | 对未见对象/接触动力学会产生 model bias；MPC 成本高。 |

## 6. 算力、硬件与风险（学习规划，不在本机部署）

| 目标 | 建议配置 | 风险 |
|---|---|---|
| 小型仿真 / 复读 | 24 GB GPU | diffusion sampling 令单步规划仍可能很慢。 |
| 视频模型后训练 | 48–80 GB GPU，多卡更现实 | 训练视频序列的显存和数据吞吐很高。 |
| 大规模预训练 | 多节点高显存 GPU | 不在当前范围。 |
| 真机 | 低层安全控制器 + 小 horizon | planner 超时必须丢弃，而非继续执行陈旧动作。 |
