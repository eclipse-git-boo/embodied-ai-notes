---
layout: page
title: "OpenVLA：把连续机器人动作变成语言 token"
description: "经典开源 VLA 基线：从视觉语言模型到 7-DoF 动作预测。"
---

> **精读定位**：这是后续阅读 π0 / π0.5 前最重要的离散动作基线。先弄清它的 token 化、归一化和部署假设，才能看出 flow matching 的必要性。

<figure class="paper-figure">
  <img src="{{ '/assets/paper-figures/openvla-02.png' | relative_url }}" alt="OpenVLA 论文第 2 页图示">
  <figcaption>论文图示摘录（PDF 第 2 页）。来源：OpenVLA, Fig. 1/2 附近；原文见下方链接。</figcaption>
</figure>

## 1. 快读卡片

| 项目 | 内容 |
|---|---|
| 论文 | *OpenVLA: An Open-Source Vision-Language-Action Model*（2024） |
| 原文 / 代码 | [arXiv:2406.09246](https://arxiv.org/abs/2406.09246) · [官方仓库](https://github.com/openvla/openvla) |
| 输入 / 输出 | 单或多相机观测、自然语言指令 → 7-DoF 末端执行器连续动作 |
| 核心选择 | 用 256 个离散 bin 表示每个动作维度，让 VLM 以 next-token prediction 学动作 |
| 对 π0.5 的价值 | 它展示了“VLM + 离散动作 token”的清晰基线，也暴露了长动作 token 串与自回归延迟的问题。 |

## 2. 动机与相关工作

机器人模仿学习常把图像编码器、语言编码器和策略头分开训练；VLA 则希望复用互联网视觉语言预训练的语义能力。OpenVLA 的问题很务实：没有开放权重、开放训练配方和可复现实验的 VLA，研究者很难确认性能差异来自模型、数据还是评测。

它承接 RT-2 式的“把动作当 token”，但选择开放的 Prismatic VLM 骨干与 Open X-Embodiment 数据。与端到端连续回归相比，token 化可以直接复用语言模型的交叉熵训练；代价是每一控制步要预测多个强相关 token，误差会量化且按自回归顺序累积。

## 3. 方法：从动作向量到词表

对动作向量 \(a\in\mathbb{R}^7\)，先按训练集统计量裁剪/归一化，再将每一维映射至 256 个 bin。每个 bin 被映射到语言模型词表末端的动作 token；模型在图像、文本和既有 token 条件下预测下一个 token：

\[
p(a\mid o, x)=\prod_{d=1}^{7}p(t_d\mid o,x,t_{<d}).
\]

训练损失就是动作 token 的交叉熵；部署时反量化得到连续指令。该设计的实现关键不是公式本身，而是**训练与在线运行必须共享**：相机坐标系、动作语义（末端 delta 还是 joint delta）、夹爪编码、归一化统计量和控制频率。

<div class="method-flow"><span>图像 + 指令</span><b>→</b><span>Prismatic VLM</span><b>→</b><span>7 × action tokens</span><b>→</b><span>反量化 / 安全过滤</span><b>→</b><span>机器人</span></div>

## 4. 实验读法与论文结论

论文在 LIBERO 等基准上比较全参微调、LoRA 和不同视觉语言骨干；其最重要的实验贡献是给出可公开复跑的基线，而非声称离散 token 在所有控制问题都最好。阅读曲线时应分开看：离线 token accuracy、任务成功率、端到端延迟；前两者高并不保证真机稳定。

论文的适用边界也很明确：固定或相近相机、训练分布内的控制接口、短动作输出。对高频、长时程或多自由度平台，逐维自回归序列会成为瓶颈——这正是 FAST 与 π0 的改进方向。

## 5. 可复现性审计

| 项目 | 状态 / 你需要确认的内容 |
|---|---|
| 代码与权重 | 官方仓库公开；先使用其依赖锁定版本和官方 checkpoint。 |
| 数据 | 依赖 Open X-Embodiment / LIBERO 生态；自采数据必须转换为同一 episode、时间戳和 action schema。 |
| 最常见复现坑 | 训练集与部署端的动作归一化不一致；相机 RGB 顺序、裁剪或延迟变化；把 joint action 当成末端增量。 |
| 对 SO-ARM101 | 不可直接套用原统计量。先定义安全的 delta action、限制工作空间，并采集少量单任务闭环数据做 smoke test。 |

## 6. 算力、硬件与风险（学习规划，不在本机部署）

| 目标 | 现实配置 | 说明 |
|---|---|---|
| 只读 / 小样本 LoRA | 单卡 24–48 GB（4090 48 GB 可作为学习预算） | 采用 4-bit/QLoRA、低分辨率、梯度累积；先跑离线评估。 |
| 全参微调 7B | 推荐 4–8 × 80 GB 级 GPU | 激活、优化器状态和多相机序列使单卡方案通常不稳健。 |
| 从头预训练 | 多节点数据中心级集群 + 大规模机器人数据 | 不是个人复现目标；OpenVLA 的价值是使用已训模型再适配。 |
| 在线推理 | 24 GB 以上 GPU + 独立安全控制层 | 控制机需能处理相机、模型服务和急停；不要让模型进程直接拥有无约束执行权。 |

主要风险：量化误差、动作空间错配、视觉延迟和安全边界遗漏。对你的当前学习主线，OpenVLA 应作为**理解与对照基线**，而不是优先复现对象。
