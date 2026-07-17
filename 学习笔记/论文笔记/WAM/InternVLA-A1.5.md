---
layout: page
title: "InternVLA-A1.5：latent foresight 与可部署动作专家"
description: "A1 的升级：保留 VLM 语义，用冻结视频模型监督潜在未来。"
paper_domain: WAM
---

> **一句话结论**：A1.5 将 A1 的“生成未来视觉”改造成训练期的 latent foresight 查询：冻结视频生成器只提供动态先验，推理时丢弃视频分支，使操控策略兼顾语义、预测性与低延迟。

## 0. 快读卡片

| 项目 | 内容 |
|---|---|
| 论文 | [InternVLA-A1.5: Unifying Understanding, Latent Foresight, and Action for Compositional Generalization](https://arxiv.org/abs/2607.04988)（2026） |
| 官方实现 | [InternVLA-A-series](https://github.com/InternRobotics/InternVLA-A-series) |
| 最佳技术解读 | [官方项目页](https://internrobotics.github.io/internvla-a15.github.io/)：含模型概览、模型卡和发布入口；独立中文长文尚不足以替代代码/论文。 |
| 规模 | 论文披露：1.2M 机器人 episode + 3M 多模态样本预训练。 |

### 图谱：六类证据图

| 类别 | 证据 | 放置位置 |
|---|---|---|
| 总体/I-O/动作 | 论文 Fig. 2 | 1 |
| 数据/训练、任务/环境、主结果 | 论文 Fig. 1 | 1 |
| 消融/效率 | 论文 Fig. 4 与 action-only 推理设计 | 2、6 |

## 1. 模型原理总览：输入、输出与模块分工

| 接口 | 内容 | 作用 |
|---|---|---|
| 输入 | 图像、多模态指令、训练时未来监督与动作序列 | 当前任务语义与动态条件。 |
| 共享主干 | 原生 Qwen3.5-2B VLM | 保留视觉语言理解，并继续接收 VQA/子任务监督。 |
| foresight | 少量可学习 foresight tokens | 从共享上下文压缩、查询任务相关的未来 latent。 |
| 输出 | Flow Matching 连续 action chunk | 推理时只保留动作路径。 |

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/samples/internvla-a15-fig1-overview.png' | relative_url }}" alt="InternVLA-A1.5 的数据、训练推理架构、仿真与真实任务总览" /><figcaption><strong>论文 Fig. 1（裁切）。</strong>覆盖：数据与训练配方、任务环境、主结果。图中清楚区分训练期的视觉生成模型与推理期的 VLM/统一专家；下方同时给出仿真及真实任务证据。</figcaption></figure>

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/samples/internvla-a15-fig2-framework.png' | relative_url }}" alt="InternVLA-A1.5 的预训练 VLM、统一专家、foresight token 与动作查询框图" /><figcaption><strong>论文 Fig. 2（裁切）。</strong>覆盖：模型总体、输入输出与动作表示。预训练 Qwen3.5-2B VLM 和轻量 unified expert 只通过 full-attention 层交换上下文；foresight token 只为训练期未来预测服务，动作 query 负责连续控制。</figcaption></figure>

**输入到输出**：1）VLM 编码图像和指令；2）foresight token 从共享上下文中提取与任务相关的未来信息；3）冻结 WAN2.2-5B 视频模型在训练期提供该 latent 的监督；4）动作查询在统一专家中以 Flow Matching 预测连续 action chunk；5）部署时不加载视频分支，重新观测后滚动推理。

## 2. A1.5 相对 A1 改了什么

| 问题 | A1.5 的处理 | 代价/边界 |
|---|---|---|
| 像素级预测易受生成误差与计算拖累 | 用 latent 查询替代从零像素生成 | latent 更难可视化诊断。 |
| 多任务训练可能损伤 VLM 语义 | 原生 VLM 保留 VQA、子任务预测训练 | “保留”仍应以组合泛化实验验证。 |
| 视频世界模型不利于实时控制 | 训练时冻结 WAN2.2-5B，推理时丢弃视频支路 | 这减少部署负担，不等于已消除动作延迟与安全问题。 |

训练期目标可概括为：动作损失加上未来 latent 对齐损失。

$$
\mathcal{L}=\mathcal{L}_{\mathrm{FM}}+\lambda\,\lVert q_{\phi}(h_{\mathrm{ctx}})-z_{\mathrm{future}}^{\mathrm{video}}\rVert_2^2.
$$

$q_\phi$ 是 foresight token 查询器，$h_{ctx}$ 是多模态上下文，$z_{future}^{video}$ 是冻结视频生成器给出的目标 latent；$\lambda$ 为权重。此式是阅读性抽象，具体 token 布局和损失请以论文/实现为准。

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/samples/internvla-a15-fig4-foresight.png' | relative_url }}" alt="InternVLA-A1.5 通过 foresight query 和冻结 WAN 视频模型学习未来潜变量的机制" /><figcaption><strong>论文 Fig. 4（裁切）。</strong>覆盖：消融/效率的机制依据。梯度通过冻结视频模型的条件路径回传到 foresight token，而部署时不运行视频生成器；这正是 A1.5 相对 A1 的关键效率取舍。</figcaption></figure>

## 3. 官方代码：训练、微调、评估怎样组织

官方仓库将 A1.5 作为当前主线，A1 被保留在 `InternVLA-A1` 分支。关键入口如下：

| 目的 | 官方路径/入口 | 阅读重点 |
|---|---|---|
| 预训练 | `launch/internvla_a15_pretrain.sh` | 发现 `data/a1` 下的 LeRobot 数据；应用 `configs/weight_rules_pretrain.yaml`；使用 Qwen3.5-2B、FAST action token、WAN 辅助分支。 |
| LeRobot 微调 | `launch/internvla_a15_finetune.sh` | 先把数据转为 LeRobot v3.0，再计算 delta-action `stats.json`。 |
| 基准微调/评估 | README 指向 RoboTwin、LIBERO、LIBERO-Plus、DOMINO、SimplerEnv 指南 | 每个 benchmark 的 action convention、相机和成功判据不能混用。 |
| 真实机器人推理 | `config.inference_backend='optimized'`、`action_loss_only=True` | 跳过 WAN 视频分支，仅用于低延迟 action prediction。 |

**代码解析**：`pretrain` 是大规模数据和多目标配方；`finetune` 是把目标本体数据对齐到 LeRobot 格式；`stats.json` 决定动作反归一化，错误时模型 loss 可正常下降但机械臂会输出错误尺度；`optimized` 是推理加载裁剪，不是安全控制器。

## 4. 4090 + SO-ARM101：学习型微调路径

以下是**离线到受控验证**的学习流程，不在本机执行真机控制。

1. **锁定版本**：记录仓库 commit、CUDA/PyTorch、Qwen/transformers 补丁；先创建独立环境，不与 openpi 混装。
2. **先完成数据契约**：为每个 episode 保存相机时间戳、主从臂状态、夹爪状态、语言、动作坐标系、单位和频率；确认 SO-ARM101 关节顺序、零位、方向、限位与 A1.5 预训练本体不同。
3. **转换与审计**：转 LeRobot v3.0，计算 `stats.json`；可视化随机 50 条轨迹，检查图像—状态—动作是否同一时刻、delta action 是否有单位/符号反转。
4. **单卡小样本微调**：只尝试官方支持的参数高效/冻结策略、小 batch、短 action horizon 和小分辨率；先用离线 action prediction、仿真或回放评估，而非接到电机。
5. **独立安全门**：若未来由你自己决定进行台架测试，应先设置速度/工作空间/急停/人工在场/碰撞与夹爪力限制，并将模型进程与底层控制进程隔离；这不是论文已提供的部署保证。

## 5. 证据与 WAM Q&A

论文报告六个仿真 benchmark 及真实组合泛化；主张的证据是“保留语义 + latent foresight”带来更强的 held-out instruction binding 与长程执行。阅读时应分别看语义组合泛化、最终成功率和失败阶段，不能只看平均分。

**Q：预测的世界状态是什么？** A：任务相关未来 latent，不是部署时输出的完整未来视频。  
**Q：是否因果地由动作条件化？** A：A1.5 的控制目标是动作生成；未来 latent 是辅助条件/监督，不能据此宣称完整反事实模拟。  
**Q：怎样防漂移？** A：滚动重新观测、短动作块和安全停止逻辑仍由系统工程承担；论文并未给出通用 SO-ARM101 安全闭环证明。  
**Q：与 π0.5 的共同点？** A：都将强语义与连续动作专家结合；A1.5 的特点是借冻结视频生成器做训练期 latent foresight，而 π0.5 侧重异构知识与子任务语义。

## 6. 算力与硬件需求（学习与部署规划）

| 场景 | 模型/设置 | GPU 与显存 | 其他硬件 | 证据等级 | 限制 |
|---|---|---|---|---|---|
| 论文级预训练 | 1.2M robot episodes + 3M 多模态样本；VLM + WAN 辅助 | 完整 GPU 数/时长未披露 | 大容量高速数据存储、分布式训练环境 | 论文披露 + 未知 | 单张 4090 不可作为全量复现配置。 |
| 全参微调 | 2B VLM + 优化器、激活、序列 | 4090 24GB 不应假定可行 | ≥64GB RAM、NVMe | 显存下界估算 | 仅 BF16 权重约 4GB，不含训练状态。 |
| 参数高效微调 | 冻结大部分主干、短序列/小 batch | 4090 24GB 可尝试离线小实验 | 建议 1TB+ NVMe、数据备份 | 显存下界估算 | 取决于具体实现是否提供 LoRA/量化支持；先看 OOM 与离线指标。 |
| 推理/评估 | `optimized` + `action_loss_only` | 4090 24GB 有机会承载 | GPU 与控制 PC 建议逻辑隔离 | 官方实现要求 + 未知 | 官方优化后端不等于 SO-ARM101 实时/安全认证。 |

显存最敏感的项依次是：训练优化器与梯度、视觉/动作序列激活、视频辅助分支、batch size；KV cache 主要影响长上下文推理。SO-ARM101 侧还需相机/状态时间同步、独立急停和控制限幅；官方仓库并未发布针对该本体的标定、动作适配或成功率。

## 7. 来源

- **论文事实：**[论文](https://arxiv.org/abs/2607.04988)。
- **官方代码：**[InternVLA-A-series README](https://github.com/InternRobotics/InternVLA-A-series)。
- **辅助解读：**[官方项目页](https://internrobotics.github.io/internvla-a15.github.io/)用于理解模型卡与发布入口；独立长文尚不足以替代论文。
- **个人推断：**4090 与 SO-ARM101 的资源/风险说明仅为离线学习规划，不是论文或仓库给出的成功承诺。
