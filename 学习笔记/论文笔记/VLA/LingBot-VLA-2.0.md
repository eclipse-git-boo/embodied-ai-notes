---
layout: page
title: "LingBot-VLA 2.0：统一 55 维动作与预测动力学"
description: "从基础模型走向 20 种本体、全身动作和真实后训练。"
---

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/lingbot-vla-2-02.png' | relative_url }}" alt="LingBot-VLA 2.0 论文图示"><figcaption>论文图示摘录（PDF 第 2 页）。来源：[LingBot-VLA 2.0](https://arxiv.org/abs/2607.06403)。</figcaption></figure>

## 1. 快读卡片

| 项目 | 内容 |
|---|---|
| 论文 | *From Foundation to Application: Improving VLA Models in Practice*（2026） |
| 原文 / 代码 | [arXiv:2607.06403](https://arxiv.org/abs/2607.06403) · [官方仓库](https://github.com/Robbyant/lingbot-vla-v2) |
| 数据 | 约 60,000 小时：50,000 小时机器人轨迹（20 种构型）+ 10,000 小时第一视角人类视频。 |
| 核心改动 | 55 维统一动作表示、稀疏 MoE action expert、DINO-Video / 深度的双查询蒸馏、未来预测辅助任务。 |

## 2. 动机与相关工作

第一代多集中于双臂操作；移动底盘、腰头、灵巧手等全身自由度让跨本体动作对齐变得更难。LingBot-VLA 2.0 的立场是先统一状态/action 表达，再让 MoE 同时存储通用与专门模式，并用未来视觉/几何预测增加时序和空间约束。

它仍是 VLA，不等同于 LingBot-VA：此处未来预测是辅助的 VLA 表征/训练信号；VA 系列则显式把视频动力学与动作生成建成世界模型。

## 3. 方法：55 维、MoE 与双查询蒸馏

统一向量覆盖双臂关节、末端位姿、夹爪、手指、腰、头、移动底盘及保留维。对某一机器人不存在的维度，需要 feature mapping、mask 和规范化统计量共同定义；不能简单填零后忽略损失。

MoE action expert 用稀疏路由在固定激活计算量下分离共享/专门能力。双查询蒸馏把当前与未来感知 query 接到视觉文本 token，从 LingBot-Depth 和 DINO-Video 获得几何与时序教师信号。

<div class="method-flow"><span>异构本体数据</span><b>→</b><span>55D canonical action + mask</span><b>→</b><span>VLM / MoE action expert</span><b>→</b><span>深度+视频蒸馏</span><b>→</b><span>后训练策略</span></div>

## 4. 实验结论

作者在 GM-100 双臂任务和长时程移动操作上比较 π0.5 等基线。报告中，冰箱分类任务 ID 为 77.1/60.0（进度/成功率），OOD 为 37.0/13.3；后者的明显下降本身很重要：多本体预训练不会消除分布外难题。官方部署说明称，在 RTX 4090D、10 次去噪步下单次推理约 130 ms；这是官方特定实现，不是对普通 4090 的保证。

## 5. 可复现性审计

| 项目 | 要点 |
|---|---|
| 公开性 | 代码与 6B native-depth checkpoint 公开；还依赖 Qwen3-VL、MoGe、LingBot-Depth、DINO-Video 等权重。 |
| 数据接口 | 官方明确要求 LeRobot v2.1/v3.0、机器人 YAML 映射和 norm stats JSON。 |
| 最关键验证 | 先 open-loop eval；逐维检查 55D mapping、缺失维 mask 与反归一化。 |
| 限制 | 60k 小时数据、20 本体和教师模型并非个人全量复刻范围。 |

## 6. 算力、硬件与风险（学习规划，不在本机部署）

| 目标 | 建议配置 | 注意 |
|---|---|---|
| 6B checkpoint 推理 / 受限后训练 | 48 GB 单卡起，NVMe 2 TB | 官方依赖 Python 3.12 / PyTorch 2.8；先用容器或隔离环境。 |
| 全参后训练 | 建议 8 × 80 GB 级 GPU | MoE、视觉序列和教师蒸馏使显存/通信需求显著增加。 |
| 从头训练 | 多节点集群、60k 小时数据、多个教师权重 | 不作为学习复现目标。 |
| 机器人接入 | 强制 canonical action 的限幅、缺失维 mask、急停 | 55D 映射若列顺序错位，比普通 7D 更难发现且危险。 |

对当前 π0.5 目标的取舍：借鉴其 LeRobot mapping / norm stats 检查表，不要先引入 55D、MoE 或深度蒸馏复杂度。
