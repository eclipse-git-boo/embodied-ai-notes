---
layout: page
title: VLX-Go：短视野连续 waypoint 的开源研究资料卡
---

# VLX-Go：短视野连续 waypoint 的开源研究资料卡

> **来源状态：开源研究资料卡，不是正式论文精读。**当前可核验材料是官方仓库与 Hugging Face 社区技术博客；未检索到对应 arXiv/正式论文，因此不会补写未披露的模型结构、数据规模或数值结果。

## 快速卡片

| 项目 | 内容 |
|---|---|
| 官方仓库 | [om-ai-lab/VLX-Go](https://github.com/om-ai-lab/VLX-Go) |
| 最佳技术解读 | [OMLab 的 Hugging Face 技术博客](https://huggingface.co/blog/omlab/vlx-go)，完整说明短视野 waypoint、外部控制器和离线/在线学习叙事。 |
| 公开主张 | 0.6B 轻量视觉语言 planner；以最近单目帧和语言指令预测局部 waypoint，控制与安全由外部模块处理。 |

## 1. 模型原理总览：输入、输出与模块分工

<figure class="paper-figure">
  <img src="{{ '/assets/paper-figures/vln/vlx-go-architecture.svg' | relative_url }}" alt="依据 VLX-Go 官方资料重绘的输入模型输出框图" />
  <figcaption><strong>依据官方仓库/技术博客重绘。</strong>VLX-Go 的边界是短视野视觉—语言 planner：它输出局部连续 waypoint，不直接承担平台控制与安全。</figcaption>
</figure>

| 模块 | 可核验职责 | 不可从公开资料推出的结论 |
|---|---|---|
| 视觉语言 planner | 融合最近单目帧与自然语言 | 完整长程语义地图、精确网络层数 |
| waypoint 表示 | 短时域的局部连续路径 | 对所有机器人都可直接执行 |
| 外部 controller | 处理运动学、安全与平台差异 | VLX-Go 自带碰撞保证 |

### 从输入到输出的 4 步

1. 缓存最近图像和当前语言目标；
2. 0.6B 级 planner 形成局部视觉—语言条件；
3. 预测下一段连续 waypoint；
4. 外部控制器转换并执行。后两步之间的标定与限幅是工程责任，不应被省略。

### 图谱：六类证据图

| 类别 | 唯一图 | 公开证据状态 |
|---|---|---|
| 模型总览、I/O、动作表示 | 架构重绘 | 官方仓库/博客描述 |
| 数据/训练、效率 | 训练—反馈循环重绘 | 博客叙述，具体配方未公开 |
| 任务/环境、主结果、消融边界 | 评测信息边界图 | 指标名称可见，未发现正式论文结果表 |

## 2. 训练叙事与真正需要核对的字段

<figure class="paper-figure">
  <img src="{{ '/assets/paper-figures/vln/vlx-go-training.svg' | relative_url }}" alt="VLX-Go 官方技术博客描述的离线学习与仿真反馈循环重绘" />
  <figcaption><strong>依据官方技术博客重绘。</strong>博客描述先从离线轨迹学习，再利用仿真在线反馈；它没有替代公开的论文级训练配方与 ablation。</figcaption>
</figure>

真正应写进数据卡的是：相机帧率、历史窗口、waypoint horizon、坐标系、语言模板、仿真反馈规则、控制器接口与失败 episode。缺任一项都可能造成“loss 降了、闭环却失败”。这里特别要抵制把“短 horizon”理解成弱点：它是将高层规划与底层控制解耦的选择，但需要外部重规划补足远程记忆。

## 3. 指标与资料边界

<figure class="paper-figure">
  <img src="{{ '/assets/paper-figures/vln/vlx-go-evaluation.svg' | relative_url }}" alt="VLX-Go 公开资料提及的指标和未披露边界重绘" />
  <figcaption><strong>依据官方技术博客重绘。</strong>资料提及 EVT Bench 与 SR/TR/CR，但未检索到可作为论文结论引用的正式结果表；故本笔记不填造数值或消融结论。</figcaption>
</figure>

SR、TR、CR 即使同名也取决于终止条件、障碍定义、目标运动和试验次数。对于短视野模型，还应另看轨迹平滑度、重规划延迟、目标暂时遮挡后的恢复、控制器造成的偏差，而不是只看单次成功。

## 4. 与 VLN 主线的关系

VLX-Go 更像 VLN 系统的“局部执行层候选”：语言与视觉给出方向，连续 waypoint 接到平台控制。它未公开证明自己解决了长程 route memory、frontier exploration 或复杂语言歧义；这些分别是 OmniNav、InternVLA-N1、Qwen-RobotNav 读起来应对照的缺口。

## 5. 算力与硬件需求（学习与部署规划）

| 场景 | 公开信息 | 学习规划（非官方最低配置） |
|---|---|---|
| 全量训练 | 公开材料称 0.6B 轻量模型，但未披露全量数据、GPU-hours、正式配方 | 不能据此估算“可复现全训”；视频数据、仿真和存储仍需多卡/高速盘规划。 |
| 微调 | 模型/数据入口在仓库仍标为 coming soon 的部分应以最新 README 为准 | 待权重与脚本齐备后，优先做 24–48GB 显存的 LoRA 小样本离线验证。 |
| 推理评测 | 无官方硬件下限 | 24GB 级显存是离线短视频验证的保守起点；不要进行真机控制。 |

## 6. VLN 阅读 Q&A

**Q：为什么把控制器留在模型外？** A：不同平台的动力学、安全限幅和控制频率差异很大；分层能降低耦合，但接口误差会成为主风险。

**Q：短视野如何完成长程任务？** A：必须由外部目标更新、记忆或高层 planner 反复重设局部目标；当前公开资料没有证明完整长程方案。

**Q：看哪些泛化风险？** A：未见场景、相机高度改变、运动模糊、语言歧义、控制延迟和目标短暂遮挡。

## 来源账本

- **官方资料：**[VLX-Go GitHub](https://github.com/om-ai-lab/VLX-Go)、[OMLab 技术博客](https://huggingface.co/blog/omlab/vlx-go)。
- **资料状态：**未找到对应正式论文，故不将该项目材料伪装为论文实验结论。
- **个人推断：**硬件建议用于学习和离线评测规划。
