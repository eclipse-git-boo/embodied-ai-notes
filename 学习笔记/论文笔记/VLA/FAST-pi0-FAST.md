---
layout: page
title: "FAST / π0-FAST：快速动作 token 化"
description: "以频域编码、分块解码和 token 合并降低 VLA 动作自回归开销。"
---

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/fast-pi0-fast-02.png' | relative_url }}" alt="FAST 论文图示"><figcaption>论文图示摘录（PDF 第 2 页）。来源：[FAST / π0-FAST](https://arxiv.org/abs/2501.09747)。</figcaption></figure>

## 1. 快读卡片

| 项目 | 内容 |
|---|---|
| 论文 | *FAST: Efficient Action Tokenization for Vision-Language-Action Models*（2025） |
| 原文 / 代码 | [arXiv:2501.09747](https://arxiv.org/abs/2501.09747) · [官方实现](https://github.com/Physical-Intelligence/fast) |
| 核心问题 | 离散 action token 序列太长，既慢又有逐 token 误差累积。 |
| 核心答案 | 将时序动作转换、量化并压缩为更短 token 序列；π0-FAST 把该 token 化接入 π0 家族。 |

## 2. 动机与相关工作

OpenVLA 类方法按“时间 × 动作维”发 token，动作频率、自由度或 chunk 一增大，LLM 解码长度随之膨胀。单纯增大 bin 或缩小 chunk 会分别损失精度或时间一致性。FAST 的观点是：机器人轨迹中低频/结构性变化可压缩，tokenizer 本身应为控制效率而设计。

## 3. 方法：编码、量化、分块

FAST 对一段归一化动作做变换编码，利用频率域/分组表征减少冗余；随后量化成 token，并通过 token 合并与分块策略降低序列长度。解码端将 token 恢复为连续 chunk。论文把“重建误差”和“控制速度”一起看：低 token 数不是目的，能在真实控制频率下保留可执行轨迹才是目的。

<div class="method-flow"><span>连续 action chunk</span><b>→</b><span>变换编码</span><b>→</b><span>量化 + token 合并</span><b>→</b><span>短 token 串</span><b>→</b><span>VLA 解码 / 反变换</span></div>

对 π0-FAST 而言，名称容易误解：π0 的主路径是 flow action generation；FAST 是一套可用于动作 token 化与高效解码的配套方法。阅读时应在代码中确认实际选择的 policy/config，而不要把两者简单等同。

## 4. 实验结论与边界

论文在 VLA 任务上报告 token 数和推理吞吐改善，同时比较控制效果。应重点检查三类数字：压缩比、端到端控制频率、任务成功率。若只报告 token 减少，无法说明 token 化是否破坏接触阶段的微动作；若只报告平均延迟，也可能掩盖长尾卡顿。

适用边界：FAST 需要与特定动作维度、chunk 长度、归一化统计量共同训练/拟合。换机械臂、换控制频率或把夹爪从二值改为连续后，应重新建立 tokenizer 与验证集。

## 5. 可复现性审计

| 项目 | 检查点 |
|---|---|
| 代码 | 先跑官方 encode/decode 单元测试，记录每维 reconstruction error。 |
| 数据 | episode 必须时间等间隔；不要把掉帧/插值错误当作动作高频信息。 |
| 评估 | 同时保存 action MAE、频率响应、控制 Hz、真机成功率。 |
| 常见坑 | 用训练集统计量误套到新臂；tokenizer 成功重建开环轨迹，却在闭环观测延迟下失效。 |

## 6. 算力、硬件与风险（学习规划，不在本机部署）

| 目标 | 建议配置 | 说明 |
|---|---|---|
| tokenizer / 离线消融 | 单卡 12–24 GB 或 CPU 可完成大量预处理 | 先做离线编码-解码可视化。 |
| 基于 checkpoint 的 LoRA | 单卡 24–48 GB | 短 token 串通常降低自回归显存与延迟，但不自动保证安全。 |
| 全参多任务训练 | 多卡 80 GB 级 | 瓶颈主要来自 VLM 和异构数据，不仅是 tokenizer。 |
| 在线控制 | 低延迟 GPU、稳定时钟、独立限幅器 | 必测 p95/p99 latency；不要只看平均速度。 |

对 π0.5 主线的直接启发是：把 **动作表示质量** 和 **在线延迟预算** 当成同一个工程问题，而不是训练完成后的优化项。
