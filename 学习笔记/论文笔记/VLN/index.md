---
layout: page
title: VLN 论文与开源资料卡
---

# VLN：视觉-语言导航

这里按“**论文精读**”与“**开源研究资料卡**”分开管理。前者有可下载的原文；后者是公开仓库或模型卡，不能把其 README 的主张误写成同行评审论文结论。

## 论文精读

- [Qwen-RobotNav：可配置的 Agentic 导航执行器](Qwen-RobotNav.html)
- [OmniNav：快慢双系统的统一导航与探索](OmniNav.html)
- [InternVLA-N1：像素目标与潜在计划的双系统 VLN](InternVLA-N1.html)

## TrackVLA / Om AI Lab VLN 演进

- [TrackVLA → OmTrackVLA → VLX-Go：跟踪导航路线演进](TrackVLA-OmAI-Lab-VLN.html)

## 建议阅读顺序

先看 OmniNav 的“记忆/前沿 + 快慢系统”，再看 InternVLA-N1 如何把高层计划变成异步的潜在 token，最后看 Qwen-RobotNav 如何把任务模式和观测预算暴露给上层 agent。再用 TrackVLA → Om AI Lab VLN 路线补齐动态跟随、短历史、局部 waypoint、控制器分层和指标边界。
