---
layout: page
title: VLA / Video-Action 论文精读
---

# VLA / Video-Action 论文精读

本区采用统一阅读顺序：**输入与动作接口 → 模型模块 → 数据/训练 → 结果与消融 → 局限 → 硬件审计 → 领域 Q&A**。所有论文图均从原论文裁切为独立证据图：模型总览、数据/训练、输入输出、任务环境、主结果、消融/规模/效率六类至少覆盖一次，同一图不会重复出现。

## π0.5 复现主线

1. [OpenVLA：离散 action token 的开源 VLA 基线](OpenVLA.html)
2. [π0：用 Flow Matching 生成连续动作](pi0.html)
3. [FAST / π0‑FAST：把高频动作压缩为可预测 token](FAST-pi0-FAST.html)
4. [π0.5：开放世界 VLA](pi05.html)
5. [Knowledge Insulating VLA：用离散知识保护连续控制](pi05-knowledge-insulating.html)
6. [Qwen‑VLA：统一操作、导航与具身理解的 DiT 动作专家](Qwen-VLA.html)

建议把前五篇作为“直接 policy 与动作表示”主线。Qwen‑VLA 增加多任务、VLN 和本体提示的统一接口，适合作为 π0.5 后的对照阅读。

## LingBot 系列：规模、统一动作与 Video‑Action

- [LingBot‑VLA：20k 小时多双臂数据与高吞吐后训练](LingBot-VLA.html)
- [LingBot‑VLA 2.0：统一 55 维动作与预测动力学](LingBot-VLA-2.0.html)
- [LingBot‑VA：因果视频—动作世界模型](LingBot-VA.html)
- [LingBot‑VA 2.0：规划器、原生视频动作预训练与前瞻推理](LingBot-VA-2.0.html)

其中 LingBot‑VA 两篇放在本区以便与 VLA 对照，但方法范式属于 Video‑Action/WAM：请重点区分“直接动作策略”与“动作后果预测是否参与决策”。

## 学习边界

本仓库面向论文阅读、离线数据处理和仿真/日志复现准备。硬件部分仅给出全量训练、微调和推理的规划信息；不在这台学习机上执行或宣称验证真实机器人部署。
