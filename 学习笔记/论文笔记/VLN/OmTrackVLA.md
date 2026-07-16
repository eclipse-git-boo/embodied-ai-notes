---
layout: page
title: OmTrackVLA：视觉导航与跟随的开源研究资料卡
---

# OmTrackVLA：视觉导航与跟随的开源研究资料卡

> **来源状态：开源研究资料卡，不是正式论文精读。**截至本笔记整理时，项目自引是 GitHub 仓库；因此以下结论只归因于官方 README、训练脚本/模型卡，不能表述为同行评审论文结论。

## 快速卡片

| 项目 | 内容 |
|---|---|
| 官方仓库 | [om-ai-lab/OmTrackVLA](https://github.com/om-ai-lab/OmTrackVLA) |
| 模型卡 | [omlab/OmTrackVLA-0.6B](https://huggingface.co/omlab/OmTrackVLA-0.6B) |
| 最佳技术解读 | [官方 README](https://github.com/om-ai-lab/OmTrackVLA)；它是当前最完整的公开技术说明。未检索到可核验的对应论文/独立长文。 |
| 公开主张 | 0.6B 开源 VLA，以单目视频与语言指令预测短视野 waypoint，用于视觉导航与跟随。 |

## 1. 模型原理总览：输入、输出与模块分工

<figure class="paper-figure">
  <img src="{{ '/assets/paper-figures/vln/omtrackvla-architecture.svg' | relative_url }}" alt="依据 OmTrackVLA README 重绘的模型信息流" />
  <figcaption><strong>依据官方 README 重绘。</strong>视频历史和语言条件经视觉语言主干形成表征，再由 waypoint 头给出局部轨迹；外部控制器不属于该模型本身。</figcaption>
</figure>

| 输入/输出 | 公开可核验内容 | 应追问什么 |
|---|---|---|
| 输入 | 单目视频历史、当前观测、自然语言目标 | 相机频率、坐标系、缺帧如何处理？ |
| 输出 | 可执行的短视野 waypoint | horizon、单位、速度与朝向是否与控制器同义？ |
| 主干 | README/模型卡中的 Qwen 系列 VLM 路线 | 哪些层可训练、视觉 token 如何采样？ |
| 控制 | 仓库将运动/安全留给外部层 | 不能据此声称碰撞安全或真机闭环性能。 |

### 从输入到输出的 4 步

1. 取视频历史和当前帧，连同文本目标组成条件；
2. VLM 形成时间—语义特征；
3. waypoint 头预测未来一小段局部路径；
4. 外部平台将路径转为实际运动命令。学习机只应做离线数据/模型分析。

### 图谱：六类证据图

| 类别 | 唯一图 | 说明 |
|---|---|---|
| 模型总览、I/O、动作表示 | 架构重绘 | 基于官方 README，不虚构未披露层细节 |
| 数据/训练 | 训练接口重绘 | 公开字段与 mask 语义 |
| 任务/环境、主结果、效率/局限 | EVT-Bench 指标重排 | README 披露的任务分组与数值边界 |

## 2. 数据接口与训练信号

<figure class="paper-figure">
  <img src="{{ '/assets/paper-figures/vln/omtrackvla-data.svg' | relative_url }}" alt="OmTrackVLA 官方 README 中 history horizon 和掩码监督字段的重绘图" />
  <figcaption><strong>依据官方 README 重绘。</strong>样例配置公开了 `history=31`、`horizon=8`、`dt=0.1`，以及视频、指令、未来 waypoint、速度命令和掩码等字段。</figcaption>
</figure>

最值得学习的是接口纪律：历史长度、预测长度和采样间隔不是普通超参数，它们定义了“一个 label 到底表示多久的未来”。README 还给出 masked waypoint loss 的训练线索；mask 只能排除缺失/无效标签，不能充当碰撞、舒适度或跟踪安全的监督。

## 3. 结果怎样读：SR 不够

<figure class="paper-figure">
  <img src="{{ '/assets/paper-figures/vln/omtrackvla-evaluation.svg' | relative_url }}" alt="OmTrackVLA EVT-Bench 的 STT DT AT 指标边界重绘" />
  <figcaption><strong>依据官方 README 重排。</strong>项目在 EVT-Bench 分 STT、DT、AT 报告 SR/TR/CR。AT 的 0.6B 行为 SR 60.04、TR 73.89、CR 7.60；这些是仓库披露，不是经由本文额外复核的论文结果。</figcaption>
</figure>

SR 是成功率，TR/CR 的具体定义和试验协议必须回到 benchmark；即便 README 表示 AT SR 高于 TrackVLA，也应同时看碰撞率和测试场景。对“跟随”任务尤其要追问：目标丢失、急转、遮挡时的恢复策略在哪里，是否有长时序 drift 指标。

## 4. 复现与风险边界

- 本项目是局部 waypoint 模型，不能从短视野输出直接推断其拥有全局拓扑地图或长程语言规划。
- 不同轮式/足式平台的控制频率、镜头高度和速度上限会改变标签分布；复现需先统一坐标与时间戳。
- 公开 checkpoint 能跑通不等于模型在动态人群、狭窄空间或真机安全场景可用。

## 5. 算力与硬件需求（学习与部署规划）

| 场景 | 官方披露 | 学习规划（非官方最低配置） |
|---|---|---|
| 全量训练 | 已公开 0.6B 级 checkpoint 与训练入口，未见完整 GPU-hours 披露 | 数据解码和多帧视频 IO 往往先成瓶颈；全量训练至少按多卡/高速 NVMe 规划。 |
| 微调 | README 示例含 batch、history、waypoint 配置 | LoRA/QLoRA 小样本离线试验可从 24–48GB 显存尝试；应先验证坐标归一化与 mask。 |
| 推理评测 | 未见可泛化的官方硬件下限 | 24GB 级显存可作为离线视频评测起点；不应把离线吞吐当成安全控制频率。 |

## 6. VLN 阅读 Q&A

**Q：它有地图吗？** A：公开接口重点是视频历史到短轨迹，未披露完整地图模块；不要把“历史帧”直接等同空间记忆。

**Q：为什么要看 CR？** A：成功到达但持续碰撞或过于激进的行为不满足导航质量；SR 单独不足以评价。

**Q：何处最易复现失败？** A：时间间隔、相机内外参、waypoint 坐标系、速度命令含义和无效 label 的 mask。

## 来源账本

- **官方资料：**[GitHub README](https://github.com/om-ai-lab/OmTrackVLA)、[Hugging Face 模型卡](https://huggingface.co/omlab/OmTrackVLA-0.6B)。
- **关联论文（非本项目论文）：**[TrackVLA](https://arxiv.org/abs/2505.23189)，仅作方向背景，不将其结果归给 OmTrackVLA。
- **个人推断：**硬件建议是离线学习规划。
