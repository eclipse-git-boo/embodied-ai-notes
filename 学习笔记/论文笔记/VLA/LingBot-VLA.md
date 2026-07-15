---
layout: page
title: "LingBot-VLA：务实的大规模多本体 VLA"
description: "20,000 小时、9 种双臂构型的开放 VLA 技术报告。"
---

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/lingbot-vla-02.png' | relative_url }}" alt="LingBot-VLA 论文图示"><figcaption>论文图示摘录（PDF 第 2 页）。来源：[LingBot-VLA 技术报告](https://arxiv.org/abs/2601.18692)。</figcaption></figure>

## 1. 快读卡片

| 项目 | 内容 |
|---|---|
| 论文 | *A Pragmatic VLA Foundation Model*（2026） |
| 原文 / 代码 | [arXiv:2601.18692](https://arxiv.org/abs/2601.18692) · [官方仓库](https://github.com/Robbyant/lingbot-vla) |
| 数据规模 | 约 20,000 小时真实数据、9 种常见双臂构型。 |
| 开放资源 | Apache-2.0 代码、4B / 4B-Depth checkpoint、后训练配方。 |
| 一句话 | 重点不是新奇的单一损失，而是将数据、训练吞吐、多本体评测和可操作代码结合成一套工程化 VLA 基线。 |

## 2. 动机与相关工作

VLA 的性能常被“数据量、机器人本体和训练代码效率”共同决定。只报告模型结构却不公开数据接口/训练路径，个人很难复验。LingBot-VLA 把问题定义为实用性：在多双臂平台之间获得可后训练的基础策略，同时让数据加载和训练吞吐足够高。

它与 OpenVLA 相同地从开放 VLM/机器人数据生态出发，但面向更多双臂构型和深度线索；与 π0 相比，重点更偏工程配方和大规模异构数据，而非用 flow matching 重定义动作生成。

## 3. 方法与实现抓手

模型以视觉语言骨干连接动作专家，提供无深度与深度蒸馏两种 4B checkpoint。跨机器人训练的关键并非“直接拼数据”，而是统一样本字段、相机/状态有效性、动作表示和归一化，再在训练中保留本体差异。

<div class="method-flow"><span>多臂真实轨迹</span><b>→</b><span>筛选 / 统一 schema</span><b>→</b><span>VLM + action policy</span><b>→</b><span>4B checkpoint</span><b>→</b><span>本体后训练</span></div>

官方仓库注明 Python 3.12.3、PyTorch 2.8、CUDA 12.8，并给出 4B / 4B-Depth 下载。对于复现，版本锁定和数据字段映射比从论文手写网络更优先。

## 4. 实验与论文结论

技术报告在三种平台、100 个任务的设置下比较多种 VLA；作者强调每任务使用 130 条后训练 episode 的一般主义评估。论文还报告在 8 GPU 设置下每卡 261 samples/s、相对现有 VLA 代码库 1.5–2.8× 的吞吐提升。该数字是特定硬件/数据管线结果，不能直接外推到单卡或自己的相机数据。

## 5. 讨论与可复现性审计

| 维度 | 结论 |
|---|---|
| 强项 | 开源 checkpoint、代码与明确依赖；多本体后训练有现实参考价值。 |
| 论文边界 | 20,000 小时真实数据的采集/清洗成本远超个人；公开 checkpoint 不代表能从零复刻预训练。 |
| 复现入口 | 用 4B checkpoint 做 LeRobot 格式后训练；先做 action/state/image mapping 与 norm stats。 |
| 对 π0.5 的启示 | 多本体泛化先是数据与接口问题；先完成单本体可靠闭环，再增加本体。 |

## 6. 算力、硬件与风险（学习规划，不在本机部署）

| 目标 | 建议配置 | 风险 |
|---|---|---|
| 4B 推理/小规模后训练 | 48 GB 单卡可作为起点；Linux + 兼容 CUDA | 官方版本较新，驱动/PyTorch/FlashAttention 匹配是第一风险。 |
| 全参后训练 | 4–8 × 80 GB 级 GPU + 高速 NVMe | 多相机序列、优化器状态和数据吞吐会卡住训练。 |
| 从头预训练 | 8+ 高显存 GPU、20k 小时级数据与数据工程 | 不适合个人学习复现。 |
| 真机 | 独立急停、限位、相机标定、时间戳日志 | 不同双臂 action mapping 错误可直接造成危险动作。 |

学习优先级：把其数据 schema / 后训练路径作为参考；π0.5 仍是当前主线。
