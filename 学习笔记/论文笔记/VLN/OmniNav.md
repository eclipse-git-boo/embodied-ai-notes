---
layout: page
title: OmniNav：快慢双系统的统一导航与探索
---

# OmniNav：快慢双系统的统一导航与探索

> 核心问题：一套导航模型如何既快速生成可执行局部 waypoint，又在陌生环境中保留足够的空间—语义记忆来找目标？

## 快速卡片

| 项目 | 内容 |
|---|---|
| 论文 | [OmniNav: A Unified Framework for Prospective Exploration and Visual-Language Navigation](https://arxiv.org/abs/2509.25687) |
| 官方项目/代码 | [项目页](https://astra-amap.github.io/omninav.github.io/) / [amap-cvlab/OmniNav](https://github.com/amap-cvlab/OmniNav) |
| 最佳技术解读 | [官方项目页](https://astra-amap.github.io/omninav.github.io/) 是目前最完整的一手技术说明，含快慢系统、任务范围和演示；未检索到质量更高且可核验的独立长文。 |
| 论文事实 | 统一 instruction/object/point goal 与 frontier exploration；快系统输出 5 个连续空间 waypoint，论文报告真机闭环最高 5 Hz。 |

## 1. 模型原理总览：输入、输出与模块分工

| 输入/输出 | 内容 | 意义 |
|---|---|---|
| 快系统输入 | 当前任务/子任务、短历史图像、坐标 token | 低延迟局部避障与路径生成 |
| 慢系统输入 | 长历史、候选 frontier、图像记忆 | 决定“下一处值得去哪里” |
| 共享表征 | 文本、坐标、图像 token 与 KV cache | 保持快慢判断使用相近语义上下文 |
| 输出 | 5 个 $(x,y,\sin\theta,\cos\theta,c)$ waypoint | $c$ 是到达/完成标志，便于接口区分继续与停止 |

<figure class="paper-figure">
  <img src="{{ '/assets/paper-figures/vln/omninav-architecture.png' | relative_url }}" alt="OmniNav 快思考与慢思考系统框图" />
  <figcaption><strong>论文 Fig. 1。</strong>上半部为共享 VLM + 条件 diffusion 的快速 waypoint 生成；下半部为利用 frontier 与图像记忆的慢速全局规划。</figcaption>
</figure>

| 模块 | 作用 | 常见误读 |
|---|---|---|
| Fast Thinking | 从短上下文与子目标直接生成局部连续 waypoint | 不是只按坐标直走；仍利用视觉来绕障/对准 |
| Slow Thinking | 从候选前沿和记忆挑高层子目标 | 不是逐控制步的实时控制器 |
| Memory / frontier | 把历史图像与姿态连接到未知边界 | 不是无误差的全局语义地图 |
| Diffusion policy | 非自回归地产生 waypoint 序列 | 不等同于已解决执行安全 |

### 从输入到输出的 5 步

1. 将任务语言、点坐标和图像转成统一 token；
2. 快系统以短历史和当前子任务编码局部条件；
3. 若目标未知，慢系统对前沿对应的历史图像做语义/空间推理；
4. 慢系统选择高层子目标，快系统反复生成 5 个局部 waypoint；
5. 平台外部控制层执行局部轨迹并更新记忆。本文不在学习机上操作真实机器人。

### 图谱：六类证据图

| 类别 | 唯一证据 | 放置 |
|---|---|---|
| 模型总览、I/O、动作表示 | Fig. 1 | 本节 |
| 任务/环境、空间记忆 | Fig. 2 | 第 3 节 |
| 数据/训练 | Fig. 3（论文数据组成） | 第 4 节 |
| 主结果、消融/效率 | 论文 benchmark 与 5 Hz 报告 | 第 5 节 |

## 2. 为什么用快慢双系统

单一自回归动作模型常在长上下文和低延迟之间冲突。OmniNav 把“在眼前避开障碍、走向子目标”交给快系统，把“还没看见目标时该探索哪个未知区域”交给慢系统。快系统以条件 flow matching 预测未来 $H=5$ 个 waypoint：

$$
\mathbf w_t^{(i)}=(x^{(i)},y^{(i)},\sin\theta^{(i)},\cos\theta^{(i)},c^{(i)}),\qquad i=1,\ldots,5.
$$

论文将真值 waypoint 与噪声线性插值，学习条件向量场；推理用 5 步 Euler 积分。重点不是死记公式，而是理解：它一次处理一段连续坐标，避免离散动作 token 的量化和逐 token 延迟。

## 3. 探索时的“记忆”到底是什么

<figure class="paper-figure">
  <img src="{{ '/assets/paper-figures/vln/omninav-frontier-reasoning.png' | relative_url }}" alt="OmniNav 基于前沿和图像记忆的浴室搜索推理示例" />
  <figcaption><strong>论文 Fig. 2。</strong>图中不是直接给出动作，而是把已见视角、地图前沿和目标语义联系起来，挑选下一探索子目标。</figcaption>
</figure>

作者维护 3D 占据图以区分已探索/未知区域；每个 frontier 关联其附近、带姿态的历史图像。慢系统对这些视觉代理做语义推理，例如“找浴室”并非盲选最近 frontier，而是比较其可能通向的房间类型。这是语言 grounding 与空间记忆的耦合点；地图误差、物体遮挡和错误语义先验都可能使它选错边界。

## 4. 数据与训练

论文将导航数据和通用视觉语言数据共同训练，明确包含 caption、referring/grounding、Embodied QA 等，以增强对泛化指令与开放词汇物体的理解。读论文时需追问：这些通用数据提升的究竟是目标识别、指代消解还是路径规划？它们不能替代连续控制数据和闭环恢复数据。

## 5. 结果与边界

论文/项目页报告多基准 SOTA 及真机 5 Hz 闭环。这里不将不同 benchmark 的 SR、SPL、导航误差混为一项“总分”：SR 关心到达，SPL 还惩罚绕路，碰撞率/停止误差则反映安全和完成判定。快慢系统的核心风险也在接口处：慢系统给出语义正确但几何不可达的坐标时，快系统能否恢复取决于局部视觉、重规划频率与控制层。

## 6. 算力与硬件需求（学习与部署规划）

| 场景 | 已披露 | 学习规划（非官方下限） |
|---|---|---|
| 全量训练 | 论文未公开可复现 GPU-hours/完整集群配方 | 需要多卡训练、仿真数据生成、3D 地图与视频存储；不建议个人机从零复现。 |
| 微调 | 以官方仓库版本为准 | 小规模 LoRA/冻结视觉主干可从 48GB 级显存起规划；多帧图像、长记忆会显著抬高显存。 |
| 离线推理 | 论文报告部署频率最高 5 Hz，但未给通用显卡门槛 | 24–48GB 显存用于离线 rollout/视频评测更现实；5 Hz 真机数字不能直接外推。 |

## 7. VLN 阅读 Q&A

**Q：为何不是直接训练一个更长上下文的 policy？** A：作者认为长记忆推理和低延迟局部控制的频率需求不同；双系统是工程折中，需用消融验证其收益。

**Q：如何衡量探索质量？** A：除 SR/SPL 外，要看首次发现目标时间、重复访问、路径长度、碰撞和在未见场景上的错误恢复。

**Q：何时会失败？** A：语言含糊、候选 frontier 的视觉代理过旧、地图漂移、目标被遮挡，以及慢系统的语义先验与实际布局不符。

## 来源账本

- **论文事实：**[arXiv 原文](https://arxiv.org/abs/2509.25687)，Fig. 1–4 与方法/实验。
- **官方代码：**[amap-cvlab/OmniNav](https://github.com/amap-cvlab/OmniNav)。
- **辅助解读：**[官方项目页](https://astra-amap.github.io/omninav.github.io/)。
- **个人推断：**硬件规划只用于学习机选型，不是部署或安全保证。
