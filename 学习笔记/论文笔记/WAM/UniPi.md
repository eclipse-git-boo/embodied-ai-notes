---
layout: page
title: UniPi：文本引导的视频规划
---

原文：[OpenReview](https://openreview.net/forum?id=bo8q5MRcwy)；介绍：[Google Research](https://research.google/blog/unipi-learning-universal-policies-via-text-guided-video-generation/)；本地 PDF：[`WAM/论文/UniPi-2302.00111.pdf`](../../../WAM/论文/UniPi-2302.00111.pdf)。

## 核心想法

UniPi 把控制拆成两步：文本条件视频生成器先生成“完成下一高层目标时应该看到的未来”，逆动力学模型再将当前/未来图像转换为本体动作。视频成为跨不同动作空间的中间接口。

```text
当前图像 + 文本目标 → 视频计划 → inverse dynamics → 本体动作
```

## 重要局限

生成画面可信不等于其对应动作可执行；逆动力学误差会累计，视频生成延迟也难以满足高频控制。这是学习 WAM 时必须牢记的“可执行性缺口”。对 π₀.₅路线，UniPi 主要提供高层规划/低层控制解耦的思考，不是近期应复现的系统。

