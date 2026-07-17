---
layout: page
title: InternVLA-N1：像素目标与潜在计划的双系统 VLN
---

# InternVLA-N1：像素目标与潜在计划的双系统 VLN

> 一句话：System 2 低频理解语言和长历史，把“下一段应该朝哪里去”编码成像素目标/潜在计划；System 1 高频地把它变成可执行轨迹。

## 快速卡片

| 项目 | 内容 |
|---|---|
| 论文 | [InternVLA-N1 项目页与论文](https://internrobotics.github.io/internvla-n1.github.io/) |
| 官方实现 | [InternRobotics/InternNav](https://github.com/InternRobotics/InternNav) |
| 最佳技术解读 | [官方项目页](https://internrobotics.github.io/internvla-n1.github.io/)：含双系统演示、数据与 benchmark 摘要；未检索到可替代的高质量独立长文。 |
| 论文事实 | System 2 为 7B 预训练 VLM、约 2 Hz；System 1 是 diffusion policy、约 30 Hz；InternData-N1 超 53M 第一视角图像。 |

## 1. 模型原理总览：输入、输出与模块分工

<figure class="paper-figure">
  <img src="{{ '/assets/paper-figures/vln/internvla-n1-architecture.png' | relative_url }}" alt="InternVLA-N1 的 System 2、System 1 与控制器异步框图" />
  <figcaption><strong>论文 Fig. 4。</strong>这是理解全篇最关键的图：System 2 以 2 Hz 产出像素目标/潜在计划，System 1 以 30 Hz 读最新图像并产生轨迹，控制器约 200 Hz 执行。</figcaption>
</figure>

| 接口 | System 2（慢） | System 1（快） |
|---|---|---|
| 输入 | 指令、长历史 RGB、状态 | 最新 RGB、异步计划 token、局部目标 |
| 中间表示 | 像素目标，后续替换为潜在计划 token | 条件特征与未来轨迹 |
| 输出 | 中期目标、完成/停止判断 | 高频短程 trajectory |
| 目的 | 语言 grounding、长程规划 | 动态避障、局部执行 |

### 从输入到输出的 5 步

1. System 2 在长历史与语言中找下一中期视觉目标；
2. 预训练阶段以显式 pixel goal 让两个系统先对齐；
3. 联合阶段冻结/微调规划器的不同部分，将目标压成可学习 latent plan；
4. System 1 不等 System 2 每轮刷新，而是读最新观测并异步消费计划；
5. 外部低层控制器执行轨迹。本笔记仅讨论原理与离线算力规划。

### 图谱：六类证据图

| 类别 | 唯一证据 | 位置 |
|---|---|---|
| 模型总览、I/O、动作表示 | Fig. 4 | 本节 |
| 任务/环境、主结果 | Fig. 1 | 第 4 节 |
| 数据/训练、规模 | Fig. 2–3 | 第 3 节 |
| 消融/效率 | 论文双系统与异步训练实验 | 第 5 节 |

## 2. 为什么从 pixel goal 走到 latent plan

直接让 VLM 输出低层离散动作，会同时承受长序列误差和控制频率压力。InternVLA-N1 先把导航目标定义为图像平面中的 pixel goal：它和视觉 grounding 对齐、比轮速更接近“要去哪里”。但纯二维目标在遮挡或视角变化下会歧义，于是后续联合训练引入 latent plan token，使 System 1 获得比单点坐标更丰富、又无需同步等待语言模型的条件。

System 1 是多目标条件的 diffusion policy：输入当前观测与中间计划，输出候选短轨迹并处理局部变化。该分工应被理解为“规划频率”和“控制频率”解耦，不代表慢系统的语言判断永远正确。

将 System 1 的去噪训练写成阅读用的条件扩散目标，可表示为：

$$
\mathcal{L}_{\mathrm{S1}}=\mathbb{E}_{\tau,\epsilon,k}\left[\left\|\epsilon-\epsilon_\theta(\tau_k,k\mid o_t,p_t)\right\|_2^2\right].
$$

其中 $\tau$ 是专家短程轨迹，$\tau_k$ 是第 $k$ 个去噪步的带噪轨迹，$o_t$ 为最新视觉观测，$p_t$ 为 System 2 异步提供的 pixel goal 或 latent plan，$\epsilon_\theta$ 为 System 1 的噪声预测器。这个式子用于说明“计划条件如何进入局部轨迹生成”；精确的 token 与调度器配置仍应以论文/代码为准。

## 3. InternData-N1：数据为何是方法的一半

<figure class="paper-figure">
  <img src="{{ '/assets/paper-figures/vln/internvla-n1-data-overview.png' | relative_url }}" alt="InternData-N1 数据组成与指令关键词图" />
  <figcaption><strong>论文 Fig. 2。</strong>数据组成的同心环与指令关键词：不同圈层对应场景、距离、图像与指令的分布。</figcaption>
</figure>

<figure class="paper-figure">
  <img src="{{ '/assets/paper-figures/vln/internvla-n1-data-pipeline.png' | relative_url }}" alt="InternData-N1 数据生成、指令微调与过滤流水线" />
  <figcaption><strong>论文 Fig. 3。</strong>从场景资产和导航轨迹出发，经指令微调与数据过滤，形成最终训练数据。</figcaption>
</figure>

论文报告 3,154+ 场景、约 4,839.9 km 轨迹、53.5M 图像和约 0.8M 指令。数据由仿真场景、路径规划、渲染、开源多模态模型生成/改写指令与过滤组成。复现实验首先要核对轨迹是否无碰撞、相机投影能否把位置转成 pixel goal、语言是否与 clip 对齐；这些比“下载到多少图像”更决定是否可学。

## 4. 结果应怎样解读

论文跨 6 个 benchmark 报告 3%–28% 的提升，并强调超过 150m 长程规划与超过 30 Hz 的实时决策；其项目页给出 R2R Val-Unseen RGB-only 变体 SR 55.4、SPL 52.1。应区分：SR 是到达率，SPL 惩罚绕路，30 Hz 是 System 1 的推理节奏，不能直接等价为机器人真实闭环安全频率。

## 5. 局限与复现边界

- 像素目标受相机视角、遮挡和标定影响；latent token 更灵活但更难解释与调试。
- 异步系统可能遇到“计划已经过期而局部观测已变化”的不一致，需要观察重规划与停止逻辑。
- 论文的跨形态结果不免除控制器、传感器时间戳、动力学和安全层的逐平台验证。

## 6. 算力与硬件需求（学习与部署规划）

| 场景 | 已披露 | 学习规划（非官方下限） |
|---|---|---|
| 全量训练 | 含 7B VLM、1.3B world model/策略组件与 53M 级图像；未公开完整 GPU-hours | 多卡分布式训练和大容量 NVMe/对象存储是前提，不适合个人机全量复现。 |
| 微调 | 代码、模型、数据由项目页提供入口，具体版本以仓库为准 | 冻结视觉主干、只训练 adapter/策略头可从 48–80GB 显存规划；长历史、多图像、视频预测会扩大激活显存。 |
| 离线评测 | 论文报 System 1 >30 Hz，未给消费卡下限 | 24–48GB 用于离线视频/轨迹评测较合理；不要把离线吞吐外推成真机能力。 |

## 7. VLN 阅读 Q&A

**Q：它的记忆是什么形式？** A：长历史图像和语言由 System 2 处理，plan token 是跨频率接口；它不是显式拓扑图，因此要观察其在回退/重复访问时的证据。

**Q：为什么用双系统？** A：将长程语义推理与高频局部轨迹分离，降低同步等待；代价是接口失配和过期计划风险。

**Q：评测最容易遗漏什么？** A：Val-Unseen、连续控制协议、停止条件、碰撞、路径效率及相机/执行器时延。

## 来源账本

- **论文事实：**[论文/项目页](https://internrobotics.github.io/internvla-n1.github.io/) 与随附 PDF Fig. 1–4。
- **官方代码：**[InternRobotics/InternNav](https://github.com/InternRobotics/InternNav)。
- **辅助解读：**[官方项目页](https://internrobotics.github.io/internvla-n1.github.io/)。
- **个人推断：**硬件栏是学习规划，不是论文给出的最低配置。
