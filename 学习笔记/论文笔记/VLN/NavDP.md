---
layout: page
title: "NavDP：特权信息引导的 Sim-to-Real 导航扩散策略"
description: "从局部 RGB-D 观测生成多候选轨迹，再由 critic 选择安全路线。"
paper_domain: VLN
---

> **一句话结论**：NavDP 用仅在仿真可得的全局几何生成好轨迹与安全 critic 标签；部署时只输入局部 RGB-D 与导航目标，扩散策略生成候选 waypoint，critic 从中挑选更安全的路线。

## 0. 快读卡片

| 项目 | 内容 |
|---|---|
| 论文 | [NavDP: Learning Sim-to-Real Navigation Diffusion Policy with Privileged Information Guidance](https://arxiv.org/abs/2505.08712)（ICRA 2026） |
| 官方实现 | [InternRobotics/NavDP](https://github.com/InternRobotics/NavDP) |
| 最佳技术解读 | [官方项目页/论文 PDF](https://wzcai99.github.io/navigation-diffusion-policy.github.io/)；用于结合框图理解生成器与 critic 的共享编码器。 |
| 与 InternVLA-N1 的关系 | NavDP 是局部高频导航策略；InternNav 官方将其作为 InternVLA-N1 双系统的一种 System 1 配置。 |

### 图谱：六类证据图

| 类别 | 证据 | 放置位置 |
|---|---|---|
| 总体/I-O/动作 | Fig. 2 | 1 |
| 数据/训练 | Fig. 1、仿真数据引擎 | 3 |
| 任务/环境、主结果 | Fig. 1 与跨本体实验 | 4 |
| 消融/效率 | 特权 critic、real-to-sim 比例实验 | 5 |

## 1. 模型原理总览：输入、输出与模块分工

| 接口 | 内容 | 关键约束 |
|---|---|---|
| 输入 | 多帧 RGB、单帧深度、2D 相对 PointGoal；训练可扩展到多目标任务 | 深度被裁到有限范围以降低 sim-to-real 差异；目标坐标依赖机器人局部坐标定义。 |
| 共享编码 | 视觉 token + goal token 经 transformer 压缩 | 部署时不输入地图或 ESDF。 |
| 输出 | 多条未来 waypoint 轨迹，形式为相对 $(\Delta x,\Delta y,\Delta\omega)$ | 论文说明为 24 步候选轨迹；必须再经底层跟踪和碰撞/停止层。 |
| critic | 给候选轨迹打安全/价值分数 | 训练标签来自仿真的特权 ESDF，真机不可直接访问。 |

**数据流**：1）局部 RGB-D 与目标被编码；2）扩散头从噪声生成候选 waypoint；3）critic 读取共享特征与轨迹 token；4）选择较安全候选；5）外部控制器跟踪短前缀，下一轮重新观测。

## 2. 方法：生成与选择分别解决什么

扩散头保留多条“都可能可走”的路线，而 critic 用特权几何学习避开危险候选。抽象的去噪目标是：

$$
\mathcal{L}_{\mathrm{DP}}=\mathbb{E}_{\tau,\epsilon,k}\left\|\epsilon-\epsilon_\theta(\tau_k,k\mid o_{t-N:t},d_t,g_t)\right\|_2^2.
$$

$\tau$ 为真值 waypoint 序列，$\tau_k$ 为加噪序列，$o,d,g$ 分别为 RGB 历史、深度和相对目标。critic 的监督不是“路径看起来顺”，而是利用 ESDF 距离构造安全/危险对比样本；因此它能把仿真全局知识蒸馏到部署时的局部视觉表示。

## 3. 数据与特权信息

论文报告约 1,244 个场景、363.2 km 仿真导航轨迹，并称生成速度约为 2,500 trajectories/GPU/day。特权信息只在训练数据生成和标签构造中出现：全局规划产生高质量演示，ESDF 标注候选轨迹的碰撞风险。部署接口不需要建图，但这不意味着系统不再受传感器遮挡、深度失真或动态行人影响。

## 4. 与 InternVLA-N1 的发展关系

| NavDP | InternVLA-N1 |
|---|---|
| 局部 RGB-D + 目标到短轨迹 | 将语言、长历史和中期计划交给低频 System 2。 |
| diffusion + critic 负责安全候选选择 | 可把 NavDP 用作高频 System 1；也有 DualVLN 配置。 |
| 不解决复杂语言路线理解 | 解决“下一段去哪”的语义/计划接口，但仍依赖 System 1 落地。 |

所以阅读顺序应是先理解 NavDP 的局部闭环，再读 N1 的跨频率计划接口；二者不是重复模型。

## 5. 证据、局限与 VLN Q&A

论文在四足、轮式和人形本体、室内外环境中评估，并探索 Gaussian Splatting real-to-sim 数据以缩小域差异。主张的证据是无真实机器人训练的跨本体迁移；需要警惕的是，成功率并不能显示每次临近障碍时的安全裕度。

**Q：语言 grounding 在哪里？** A：NavDP 的主问题是目标条件导航，重点是局部视觉与 PointGoal，不是长自然语言 route VLN。  
**Q：空间记忆是什么？** A：没有显式拓扑图；历史 RGB token 提供短期上下文。  
**Q：停机与恢复？** A：论文的 candidate selection 不替代外部急停、速度限制、失效传感器检测或全局重规划。  
**Q：指标隐藏什么？** A：SR/SPL 可能掩盖碰撞次数、最小障碍距离、人工接管与重复采样造成的延迟。

## 6. 算力与硬件需求（学习与部署规划）

| 场景 | 假设 | GPU/显存 | 证据等级 | 主要限制 |
|---|---|---|---|---|
| 论文级数据生成/训练 | 并行仿真、轨迹生成、扩散与 critic | 论文未披露完整训练集群 | 论文披露 + 未知 | 生成速度不等于完整训练成本。 |
| 微调 | RGB-D、24-step trajectory、短历史 | 24GB 可能做离线小规模实验 | 显存下界估算 | 具体 batch、采样数、图像分辨率会改变显存。 |
| 推理 | 多候选扩散采样 + critic | 需测端到端最坏时延 | 未知 | 文献后续比较提到 NavDP 推理约 273ms，不能直接当作本系统承诺。 |

导航部署还需相机/深度传感器时钟、可靠里程计或目标坐标来源、底层控制器和独立安全停机链路；这些不是一张 GPU 能替代的。当前仓库仅作为学习资料，不执行真机导航。

## 7. 来源

- [论文](https://arxiv.org/abs/2505.08712)。
- [官方代码](https://github.com/InternRobotics/NavDP)。
- [InternNav：N1 与 NavDP 的组合说明](https://github.com/InternRobotics/InternNav)。
