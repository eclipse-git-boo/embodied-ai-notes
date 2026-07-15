---
layout: page
title: "LingBot-VA 2.0：原生视频-动作预训练"
description: "从语义视觉-动作 tokenizer 到因果 DiT、分层规划和前瞻推理。"
---

> **分类说明**：VA 2.0 是 VLA 学习的重要交叉材料，但主范式为 video-action foundation model / WAM；本页仍按用户要求放在 VLA 分类。

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/lingbot-va-2-02.png' | relative_url }}" alt="LingBot-VA 2.0 论文图示"><figcaption>论文图示摘录（PDF 第 2 页）。来源：[LingBot-VA 2.0](https://arxiv.org/abs/2607.08639)。</figcaption></figure>

## 1. 快读卡片

| 项目 | 内容 |
|---|---|
| 论文 | *Native Video-Action Pretraining for Generalizable Robot Control*（2026） |
| 原文 / 代码 | [arXiv:2607.08639](https://arxiv.org/abs/2607.08639) · [官方仓库](https://github.com/Robbyant/lingbot-va) |
| 核心主张 | 不把面向内容生成的视频模型勉强改成控制器，而是为具身场景从头做语义 visual-action tokenizer 与因果 video-action 模型。 |
| 关键部件 | 语义 tokenizer、因果 MoE DiT、多 chunk prediction（MCP）、人机共训、分层 VLM planner、Foresight Reasoning。 |

## 2. 动机与相关工作

视频生成通常重重建像素、常用双向结构；机器人闭环只能依赖过去和当前，且更关心可执行状态、接触和动作精度。VA 2.0 因而强调两个错位：视觉 tokenizer 的目标错位（重建 vs 语义/动作）和时间结构错位（双向生成 vs 因果控制）。

## 3. 方法：原生 token、因果世界动作模型

第一阶段训练 semantic visual-action tokenizer：视觉 latent 对齐冻结的语言对齐视觉基础模型，并从无标注视频抽取 latent action，使“世界状态”和“动作”处于同一语义空间。第二阶段用因果 mask 训练 video-action DiT：

\[
p_\theta(z_{1:N}\mid z_0)=\prod_{t=0}^{N-1}p_\theta(z_{t+1}\mid z_{\le t}).
\]

高层 VLM planner 分解长任务，低层稀疏 MoE 预测视觉和动作；MCP 监督多个未来 chunk；人机共训让第一视角人类视频提供任务变化；前瞻推理把生成和执行重叠并持续以最新观测 re-ground。

<div class="method-flow"><span>任务 → planner</span><b>→</b><span>语义 video-action tokens</span><b>→</b><span>因果 MoE DiT + MCP</span><b>→</b><span>未来状态 / 动作</span><b>→</b><span>异步执行 + re-ground</span></div>

## 4. 实验与论文结论

在 RoboTwin 2.0，论文报告 VA 2.0 在 clean/randomized 上平均 93.6%，高于前代 92.2%；tokenizer 消融中，语义 tokenizer 在 50 任务平均上优于 WAN2.2 VAE。作者也报告从 927 ms/chunk 经一致性蒸馏等加速降至 466 ms/chunk 的过程。应注意这些是特定模型、chunk 和硬件环境下的结果；高成功率模拟评测不能代替自己平台的安全验证。

## 5. 可复现性审计

| 项目 | 结论 |
|---|---|
| 公开资源 | 论文和 LingBot-VA 仓库；需确认 VA2 代码/权重版本是否与论文同步。 |
| 最可学部分 | 因果 mask、tokenizer 对齐目标、MCP 和“生成-执行重叠”的系统设计。 |
| 最难复刻部分 | 从头预训练的 tokenizer + DiT、人机共训的数据与大规模实验。 |
| 实验设计 | 分开消融 tokenizer、MCP、planner、异步执行；每次只改变一个变量。 |

## 6. 算力、硬件与风险（学习规划，不在本机部署）

| 目标 | 建议配置 | 风险 |
|---|---|---|
| 阅读/离线评估 | 48 GB 单卡 + 大容量 NVMe | checkpoint 和视频数据可能很大，先验证下载/版本。 |
| 后训练 | 多卡 48–80 GB 级 GPU 更现实 | 视觉序列与 MoE 使单卡极易 OOM；需要分布式稳定性。 |
| 原生预训练 | 多节点高显存集群、海量视频/动作或人类视频 | 不属于个人复现范围。 |
| 部署 | 低延迟 GPU 服务、分层安全控制、相机/机器人时钟同步 | 预测世界状态若与真实偏离，前瞻推理会放大误差；必须高频 re-ground。 |

它是未来 WAM 方向的高级阅读材料；就 π0.5 微调而言，先完成反应式闭环基线，再研究世界模型辅助。
