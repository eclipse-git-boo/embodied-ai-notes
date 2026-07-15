---
layout: page
title: π₀：Flow Matching VLA
---

原文：[官方 PDF](https://www.physicalintelligence.company/download/pi0.pdf)；代码：[Physical-Intelligence/openpi](https://github.com/Physical-Intelligence/openpi)；本地 PDF：[`VLA/论文/pi0.pdf`](../../../VLA/论文/pi0.pdf)。

## 核心问题与答案

π₀希望在复杂、灵巧、长时程操作上避免自回归动作 token 的慢推理。它保留 PaliGemma 的视觉语言主干，增加较小的 action expert，用 **flow matching** 直接生成连续动作 chunk。推理以少数积分步从噪声得到一段连续动作，因而控制频率比逐 token 生成更友好。

## 阅读时抓住三件事

| 项 | π₀做法 | 迁移时必须核验 |
| --- | --- | --- |
| 观测 | 多视角图像、语言、本体状态 | 相机数量/排序、状态单位和时间同步 |
| 动作 | 连续 action chunk | 关节/末端、绝对/增量、夹爪编码、chunk 长度 |
| 解码 | flow matching action expert | 积分步数、延迟、动作后处理与限幅 |

## 与 π₀.₅的关系

π₀.₅以 π₀为架构起点。先理解 π₀，才能理解 π₀.₅为什么保留连续 flow 动作头，又添加离散动作/语言联合监督和知识隔离。对本项目，π₀是“架构前置阅读”，不是独立部署目标。

## 主要风险

官方 base checkpoint 并不保证对任意本体零样本可用；动作后处理、norm stats 和控制模式不匹配时，即使网络输出数值正常，机械臂也会产生无意义或危险的运动。未来所有本体适配都应从离线 replay、速度/工作空间限幅开始。
