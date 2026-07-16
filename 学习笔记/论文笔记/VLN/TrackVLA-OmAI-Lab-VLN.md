---
layout: page
title: TrackVLA → OmTrackVLA → VLX-Go：跟踪导航路线演进
---

# TrackVLA → OmTrackVLA → VLX-Go：跟踪导航路线演进

> 这是一条“**正式论文 → 开源复现/缩小模型 → 轻量导航扩展**”的路线，而不是三篇同一作者团队的论文。原始正式论文叫 **TrackVLA**；OmTrackVLA 与 VLX-Go 都来自 Om AI Lab，但 TrackVLA 的论文作者单位是北京大学、Galbot 等，而非 Om AI Lab。

## 快速卡片

| 层级 | 资料状态 | 一句话定位 |
|---|---|---|
| [TrackVLA](https://arxiv.org/abs/2505.23189) | 正式论文（ICLR 2026） | 用共享视频 VLM 同时做目标识别与轨迹规划，以锚点扩散生成跟踪轨迹。 |
| [OmTrackVLA](https://github.com/om-ai-lab/OmTrackVLA) | Om AI Lab 开源项目/模型卡 | 官方明确说明其基于原始 TrackVLA 思路，提供 0.6B checkpoint、训练管线和 EVT-Bench 复现实验。 |
| [VLX-Go](https://github.com/om-ai-lab/VLX-Go) | Om AI Lab 开源项目/技术博客 | 官方明确说明其建立在 OmTrackVLA 技术方向上，扩展到轻量化短视野闭环导航。 |
| 最佳技术解读 | [TrackVLA 官方项目页](https://wsakobe.github.io/TrackVLA-web/) | 适合先了解论文的任务定义、数据/演示和基准；结构与结论以论文为准。 |

## 1. 先校正名称与谱系

“OpenTrackVLA”是 OmTrackVLA 仓库中 Hugging Face 导出类名 `OpenTrackVLAForWaypoint` 的一部分，并不是原始论文标题。可验证的关系如下：

1. **TrackVLA（论文）**提出 Embodied Visual Tracking：用自然语言指定目标，靠第一视角视频持续跟踪；
2. **OmTrackVLA（Om AI Lab）**官方 README 明确写“builds on the ideas introduced by the original TrackVLA project”，其重点是开放 0.6B 权重与训练/评测入口；
3. **VLX-Go（Om AI Lab）**官方 README 明确写“builds on the technical direction of OmTrackVLA”，将重点从 tracking 扩到 short-horizon waypoint 与闭环导航。

因此左侧目录采用“**TrackVLA / Om AI Lab VLN 演进**”，既能把三者放在一起学习，又不错误归属论文作者。

## 2. 模型原理总览：TrackVLA 的输入、输出与模块分工

| 输入/输出 | TrackVLA 论文定义 | 为什么重要 |
|---|---|---|
| 输入 | 历史/当前第一视角 RGB、描述目标外观的语言指令 | 任务不是“跟任意最近的人”，而是跟语言指定的那个目标。 |
| 识别输出 | 目标属性/问答文本 | 强制主干保留开放世界识别能力。 |
| 规划输出 | 机器人未来 waypoint/动作 | 论文的控制动作是线速度 $v$ 与角速度 $\omega$；成功要求保持约 1–3m 距离并面向目标。 |
| 训练 | 855K 识别样本 + 855K 跟踪样本 | 用多任务把“看对谁”和“往哪里走”绑在一个主干中。 |

<figure class="paper-figure">
  <img src="{{ '/assets/paper-figures/vln/trackvla-architecture.png' | relative_url }}" alt="TrackVLA 的视频编码、共享 VLM、跟踪扩散分支与识别语言分支" />
  <figcaption><strong>TrackVLA 论文 Fig. 2。</strong>视频历史、当前帧与语言先进入共享 video VLM；`[Track]` token 选择扩散轨迹分支，其他输入走语言建模分支。这是“识别—规划共用表征”的具体含义。</figcaption>
</figure>

| 模块 | 做什么 | 不应误解成 |
|---|---|---|
| Video encoder + projector | 用粗/细两种 token 表示历史与当前帧 | 为每一帧都保留相同高分辨率 token |
| 共享 LLM/VLM | 将视觉与指令融合为任务相关隐藏状态 | 直接输出底层电机信号 |
| LM head | 回答目标属性/识别问题 | 独立于规划、不会影响跟踪的附属任务 |
| Anchor-based diffusion head | 从轨迹 anchor 去噪并选轨迹 | 普通单点回归；它输出的是一段 waypoint 模式 |

### 从输入到输出的 5 步

1. 将过去滑窗帧压缩为粗 token，把当前帧保留更细 token；
2. 拼接语言与 `[Track]` 特殊 token，送入共享主干；
3. 若为识别/VQA，语言头自回归生成文本；若为跟踪，隐藏状态成为扩散动作头条件；
4. 动作头从聚类获得的轨迹 anchor 出发去噪，给出未来轨迹并评分；
5. 外部运动控制器执行局部轨迹；本文只讨论离线学习与硬件规划，不操作真实设备。

### 图谱：六类证据图

| 类别 | 唯一证据图 | 笔记位置 |
|---|---|---|
| 模型总览、I/O、动作表示 | TrackVLA Fig. 2 | 本节 |
| 数据/训练、任务/环境 | TrackVLA Fig. 1 | 第 3 节 |
| 轨迹生成效率/消融依据 | TrackVLA Fig. 3 | 第 4 节 |
| 主结果 | EVT-Bench 公开表（第 5 节） | 第 5 节 |

## 3. 为什么跟踪要把识别和规划联合训练

<figure class="paper-figure">
  <img src="{{ '/assets/paper-figures/vln/trackvla-overview.png' | relative_url }}" alt="TrackVLA 的数据、模型和鲁棒跟踪场景总览" />
  <figcaption><strong>TrackVLA 论文 Fig. 1。</strong>左侧是 embodied tracking 数据，右侧是开放世界识别数据；下方展示动态、长程和跨域跟踪环境。</figcaption>
</figure>

传统“检测器 → 规划器”流水线会把检测错误传给规划。TrackVLA 的选择是共享主干、双头解码：语言头让表征持续回答“目标是谁、长什么样”，扩散头则把同一隐藏状态变为轨迹。这种联合并不消除遮挡和误识别，而是让两种信号在训练时共同约束同一视觉语言表示。

论文在跟踪时使用 32 帧滑动窗口：历史帧为 coarse token，当前帧为 fine token。这个不对称设计是关键工程点——跟踪需要最新帧精确识别目标，而较远历史主要提供运动趋势，不能无限占用上下文。

## 4. 锚点扩散：从“许多可能轨迹”到一段可跟随路径

<figure class="paper-figure">
  <img src="{{ '/assets/paper-figures/vln/trackvla-diffusion.png' | relative_url }}" alt="TrackVLA 的锚点高斯噪声生成和锚点去噪过程" />
  <figcaption><strong>TrackVLA 论文 Fig. 3。</strong>作者先聚类训练轨迹得到 anchor，再在 anchor 附近加噪、去噪和评分；论文将其作为比普通扩散策略更快的轨迹先验。</figcaption>
</figure>

可把轨迹表示为 $\tau_i=\{(x_j,y_j,\theta_j)\}_{j=1}^{N_w}$。作者先以 K-means 得到 $M$ 个常见轨迹模式 $\{\bar\tau_i\}_{i=1}^{M}$，在每个模式附近采样噪声：

$$
\tilde\tau_i=\bar\tau_i+\epsilon_i,qquad \epsilon_i\sim\mathcal N(0,\sigma^2I).
$$

扩散头在共享 VLM 条件下同时预测去噪轨迹与该 anchor 是否接近真值的分数。直觉是：不从完全无结构噪声猜一切，而是从“跟随时常见的候选走法”出发，因此论文报告较少去噪步数和更高推理速度。需要注意，这一效率结论属于 TrackVLA 论文配方，不能自动迁移到 OmTrackVLA/VLX-Go 的不同实现。

## 5. Om AI Lab 两个项目分别改了什么

| 维度 | TrackVLA（论文） | OmTrackVLA（开源项目） | VLX-Go（开源项目） |
|---|---|---|---|
| 核心目标 | 语言指定目标的动态视觉跟踪 | 把 TrackVLA 思路做成完整开放、可训练的 0.6B stack | 将该方向扩到轻量短视野 navigation 与闭环 waypoint |
| 模型/接口 | Vicuna-7B + 识别 LM 头 + anchor diffusion | Qwen-based planner，单目视频 + 指令 → 8 waypoint | 0.6B planner，历史帧 + 当前帧 + 指令 → 短时域 waypoint |
| 数据/训练重点 | 1.7M 样本、识别和跟踪联合 | 公开 JSONL 样例、`history=31`、`horizon=8`、掩码 waypoint loss 与预缓存视觉 token | 官方描述为离线轨迹学习，再可用仿真反馈优化；checkpoint/dataset 尚标注 coming soon |
| 任务边界 | EVT：持续追随指定动态目标 | EVT-Bench 的 STT/DT/AT，强调可复现训练/评测 | 跟随、局部导航、动态避障、receding-horizon 闭环 |
| 重点改进 | 识别—规划共用表征与 anchor diffusion | **开放性 + 0.6B 可访问性**，不是宣称重写原论文理论 | **从跟随到局部导航扩展**，并将低层控制/安全明确留给 controller |

这张表最重要的结论是：OmTrackVLA 的“改进”主要是开放实现、较小模型与可复现实验栈；VLX-Go 的“改进”主要是把相同的短视野 waypoint 接口放到更广的导航闭环。两者的 README 数字应按项目披露阅读，不能反向当作 TrackVLA 论文的新实验结论。

## 6. 结果与比较边界

| 来源与设置 | STT SR | STT TR | STT CR | 正确读法 |
|---|---:|---:|---:|---|
| TrackVLA 7B（OmTrackVLA README 重列） | 85.1 | 78.6 | 1.65 | 对照行；具体协议仍回到 EVT-Bench/原论文。 |
| OmTrackVLA 0.6B（最新 checkpoint） | 81.41 | 82.77 | 5.13 | 公开 README 结果；更小模型的碰撞率不能被 SR 掩盖。 |
| VLX-Go 0.6B（STT） | 85.42 | 94.08 | 6.55 | 官方 README 结果；其模型与数据仍标注 coming soon，须等待完整可复现材料。 |

SR 是成功率，TR 是 tracking rate，CR 是碰撞率；三者必须一起读。把 TrackVLA、OmTrackVLA、VLX-Go 放一起很有价值，但它们的 backbone、训练数据、版本日期和评测实现不完全相同，不能声称严格 apples-to-apples 排名。

## 7. 局限与复现检查清单

1. 跟踪任务的成功不只是到达：需要目标身份正确、距离合适、朝向合理且不碰撞；
2. 历史帧长度、当前帧与历史帧的分辨率策略、相机时间戳和 waypoint 坐标系必须一致；
3. VLX-Go 把动力学和安全留给外部 controller，因此 planner 指标不能覆盖安全认证；
4. OmTrackVLA 与 VLX-Go 是开源项目材料，需持续核对 release/README 是否变化，不能补写未披露细节。

## 8. 算力与硬件需求（学习与部署规划）

| 场景 | 公开披露 | 学习规划（非官方最低配置） |
|---|---|---|
| TrackVLA 全量训练 | 论文披露 7B 主干、1.7M 样本、10 FPS 推理；未给完整 GPU-hours | 不适合个人机从零复训；需多卡、视频数据流水线和大容量高速存储。 |
| OmTrackVLA 微调 | 有 0.6B checkpoint、训练入口、混合精度示例 | 先做离线小样本数据/坐标对齐；24–48GB 显存可尝试 LoRA/小批量，视频 token 缓存需要 NVMe。 |
| VLX-Go 复现 | README 尚写 checkpoint/dataset coming soon | 等官方完整材料；此时只能做资料学习和接口设计，不能臆造可运行配方。 |
| 闭环评测 | 各资料的 FPS/指标不等于通用硬件下限 | 以离线仿真/视频回放为主；本学习机不执行真机控制。 |

## 9. VLN / EVT 阅读 Q&A

**Q：它算 VLN 吗？** A：它更准确是动态的 Embodied Visual Tracking，和 VLN 共享“视觉 + 语言 → 导航行为”接口，但目标是持续跟随特定移动实体，而非静态路线指令。

**Q：空间记忆在哪里？** A：TrackVLA 是 32 帧滑窗，不是显式全局地图；OmTrack/VLX 同样主要使用短历史。长程路线、回退和全局探索仍需要额外记忆或上层 planner。

**Q：为什么比较时不能只看 SR？** A：跟踪可“成功接近”却高碰撞，或短时间跟住后丢失；SR、TR、CR 与 episode 协议共同决定质量。

## 来源账本

- **论文事实：**[TrackVLA 论文](https://arxiv.org/abs/2505.23189)、[官方项目页](https://wsakobe.github.io/TrackVLA-web/)；Fig. 1–3、方法和实验。
- **OmTrackVLA 官方材料：**[GitHub README](https://github.com/om-ai-lab/OmTrackVLA)、[0.6B 模型卡](https://huggingface.co/omlab/OmTrackVLA-0.6B)。
- **VLX-Go 官方材料：**[GitHub README](https://github.com/om-ai-lab/VLX-Go)、[Om AI Lab 技术博客](https://om-ai-lab.github.io/index.html)。
- **个人推断：**硬件栏与“演进路线”的学习组织方式是笔记作者的总结，不是任一团队的官方技术声明。
