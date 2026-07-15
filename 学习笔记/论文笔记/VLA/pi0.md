---
layout: page
title: "π0：Flow Matching VLA 的动作生成范式"
description: "理解 π0 如何用连续 flow matching 代替逐 token 自回归动作。"
---

> **精读定位**：π0 是 π0.5 的直接前身。不要只记“diffusion policy”，要看清它如何把 VLM 语义条件与连续 action chunk 放入同一策略。

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/pi0-02.png' | relative_url }}" alt="π0 论文图示"><figcaption>论文图示摘录（PDF 第 2 页）。来源：[π0 原文](https://www.physicalintelligence.company/download/pi0.pdf)。</figcaption></figure>

## 1. 快读卡片

| 项目 | 内容 |
|---|---|
| 论文 | *π0: A Vision-Language-Action Flow Model for General Robot Control*（2024） |
| 原文 / 代码 | [项目页](https://www.physicalintelligence.company/research/0) · [openpi](https://github.com/Physical-Intelligence/openpi) |
| 核心贡献 | 用 flow matching 直接生成连续动作 chunk；把预训练 VLM 的语言/视觉能力与快速动作专家结合。 |
| 学习问题 | 给定最近观测和文本，如何生成多步、连续、跨本体的机器人动作？ |

## 2. 为什么不用离散动作 token

若将 \(H\) 步、\(D\) 维动作全部离散化，自回归解码长度约为 \(H\times D\)，对高频控制既慢又容易累计错误。π0 改为一次对连续动作块做条件生成：从噪声轨迹沿向量场积分到有效轨迹。这样动作维之间的相关性由网络直接建模，而不由 token 顺序强行决定。

与传统 diffusion policy 相近，π0 采用 flow matching：对数据动作 \(a\) 和高斯噪声 \(\epsilon\)，构造中间点 \(a_t=(1-t)\epsilon+t a\)，训练速度场 \(v_\theta\) 逼近从噪声到数据的方向。推理用少量 ODE / Euler 步积分。

## 3. 方法拆解

1. **语义主干**：预训练 VLM 处理图像和语言，提供物体、关系和指令条件。
2. **动作专家**：面向连续 action chunk 的生成模块，接收当前去噪时间 \(t\) 与条件表征。
3. **混合训练**：语言/视觉 token 与动作 token 共存；不同数据源可用不同 action adapter 映射到统一隐空间。
4. **闭环执行**：只执行生成 chunk 的前一部分，再读入新图像重规划，避免长期开环漂移。

<div class="method-flow"><span>观测历史 + 指令</span><b>→</b><span>VLM 条件</span><b>→</b><span>Flow action expert</span><b>→</b><span>连续 action chunk</span><b>→</b><span>短执行后重规划</span></div>

## 4. 实验与应如何解读

π0 在多种机器人形态、长时程家务与灵巧操作中展示统一策略的可行性。结果的重点不是“零样本万能”，而是：大规模异构数据加上连续生成动作，能比逐 token 策略更自然地支持高维、chunked 控制。

论文同时提醒真实系统中效果取决于数据覆盖、视觉状态估计、控制接口和闭环重规划。若只比较离线 MSE，可能看不到接触任务中动作多模态和失败恢复的差异；要报告任务成功、执行延迟和人身/设备安全触发次数。

## 5. 讨论与复现审计

| 维度 | 结论 |
|---|---|
| 优势 | 连续生成避免离散量化；chunk 天然表达时间相关；更适合高维动作。 |
| 限制 | 多步采样带来推理代价；动作定义、归一化和控制频率仍是跨本体迁移的硬约束。 |
| 代码 | openpi 是最合适的学习入口；先追踪 `pi0` policy、数据 transform、norm stats 与 action chunk。 |
| 复现顺序 | 先官方 checkpoint 离线推理 → 单任务数据适配 → 小步 LoRA/冻结视觉塔 → 真机闭环；不要直接全参训练。 |

## 6. 算力、硬件与风险（学习规划，不在本机部署）

| 目标 | 建议配置 | 风险控制 |
|---|---|---|
| π0 学习/小样本适配 | 48 GB 单卡、BF16/QLoRA、1 个相机、短 chunk | 先离线重放，确认 action 反归一化。 |
| 全参后训练 | 至少多卡 80 GB 级 GPU；高速 NVMe 数据盘 | 确保每个 batch 的本体/action mask 正确。 |
| 从头训练 | 大规模多机 GPU 与多本体数据 | 不是 π0.5 复现前置条件。 |
| 在线服务 | 独立 GPU 推理节点、同步相机时钟、硬件急停 | 设速度/工作空间/力矩限制；模型输出必须经过守护进程。 |

对你的 π0.5 目标：先把 π0 的 **flow 损失、chunk、norm stats、闭环频率** 讲清楚；它们比复刻论文规模更重要。
