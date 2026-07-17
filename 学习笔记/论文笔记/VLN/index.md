---
layout: page
title: Navigation 论文精读
---

# Navigation：视觉语言导航与基础策略

本区涵盖从局部无地图运动到长程语言条件规划的导航工作。阅读时需分清：局部轨迹生成、长程记忆、语言 grounding 与安全停止并不是同一个问题。

1. [Qwen-RobotNav：可配置的 Agentic 导航执行器](Qwen-RobotNav.html)
2. [OmniNav：快慢双系统的统一导航与探索](OmniNav.html)
3. [NavDP：特权信息引导的 Sim-to-Real 导航扩散策略](NavDP.html)
4. [InternVLA-N1：像素目标与潜在计划的双系统 VLN](InternVLA-N1.html)
5. [TrackVLA → OmTrackVLA → VLX-Go：跟踪导航路线演进](TrackVLA-OmAI-Lab-VLN.html)

建议顺序：先读 NavDP，建立“局部观测 → 多候选轨迹 → critic 选安全轨迹”的基础；再读 InternVLA-N1，理解如何将其扩展为低频语言计划与高频控制的双系统；最后阅读 OmniNav、Qwen-RobotNav 与 Om AI Lab 路线。
