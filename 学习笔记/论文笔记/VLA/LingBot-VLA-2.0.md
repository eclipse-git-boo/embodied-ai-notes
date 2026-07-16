---
layout: page
title: LingBot-VLA 2.0：统一55维动作与预测动力学
---

# LingBot‑VLA 2.0：统一 55 维动作与预测动力学

## 快速卡片

| 项目 | 内容 |
|---|---|
| 论文 | [From Foundation to Application: Improving VLA Models in Practice](https://arxiv.org/abs/2607.06403) |
| 官方实现 | [Robbyant/lingbot-vla-v2](https://github.com/Robbyant/lingbot-vla-v2) |
| 最佳技术解读 | [LingBot‑VLA 2.0 官方技术页](https://technology.robbyant.com/lingbot-vla-v2)，对数据处理、模型与发布入口的说明最完整。 |
| 核心结论 | 以约 60k 小时、20 种本体数据为底座，重做数据管线、统一 55 维动作表示，并加入预测动力学代理目标。 |

## 1. 模型原理总览：输入、输出与模块分工

| 输入 | 输出 | 关键约束 |
|---|---|---|
| 多视角观测、语言子任务、机器人状态/本体类型 | 统一 55 维动作向量的有效子空间 | 不同机器人只占用相应维度；未使用维度须被 mask，不能当作真实零动作 |

| 模块 | 作用 |
|---|---|
| 数据处理与子任务标注 | 将长视频拆成可学习的动作段和语言上下文 |
| 55 维统一动作表示 | 容纳双臂、底盘等异构控制并保留类型提示 |
| VLA policy | 在多模态条件下预测连续动作 chunk |
| predictive dynamics | 让表示对“动作后会发生什么”敏感，作为辅助训练信号 |
| MoE/训练工程 | 面向规模和吞吐的模型/实现选择，需与论文具体配置一起读 |

### 图谱：六类证据图

| 图 | 覆盖类别 | 用途 |
|---|---|---|
| Fig. 1 | 模型总览；数据/训练；主结果 | 论文从数据、架构到真实评测的总览 |
| Fig. 2 | 数据/训练；任务/环境 | 20 种本体与预训练数据覆盖 |
| Fig. 3 | 输入输出/动作表示；数据/训练 | 清洗、切片、标注、动作接口的流水线 |
| Fig. 5 | 消融/规模/效率 | 子任务动作统计，帮助解释长时程数据结构 |

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/samples/lingbot-vla2-fig1-overview.png' | relative_url }}" alt="LingBot VLA 2.0总体架构和训练评测概览" /><figcaption><strong>论文 Fig. 1（裁切）。</strong>2.0 把数据加工、统一动作、理解/动作模块与评测串在一张图中；它覆盖模型概览、训练来源与主结果语境。</figcaption></figure>

### 5 步数据流

1. 视频/状态轨迹经质量筛选和时间对齐。
2. 根据运动/停顿/交互变化生成子任务边界和文本上下文。
3. 将每个本体的控制写入 55 维向量，并保留类型和 mask。
4. VLA 用图像与语言预测动作，同时预测其潜在后果。
5. 评测/后训练时只读取该平台有效维度；坐标系、频率和末端执行器定义必须外部核对。

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/samples/lingbot-vla2-fig2-data.png' | relative_url }}" alt="LingBot VLA 2.0的多本体预训练数据" /><figcaption><strong>论文 Fig. 2（裁切）。</strong>数据覆盖多类机器人本体。多本体不是简单把样本拼接，动作维度、相机与控制频率的元数据同样是训练条件。</figcaption></figure>

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/samples/lingbot-vla2-fig3-pipeline.png' | relative_url }}" alt="LingBot VLA 2.0数据处理和动作统一管线" /><figcaption><strong>论文 Fig. 3（裁切）。</strong>数据处理流水线是复现的第一对象：若切段、语言、动作时戳任一处错位，训练 loss 可下降而闭环任务会失败。</figcaption></figure>

## 2. 方法与公式：统一表示不是抹平差异

可把不同本体动作嵌入统一张量 $a\in\mathbb{R}^{55}$，并以 $m\in\{0,1\}^{55}$ 标记有效维度：

$$
\mathcal{L}_{\mathrm{act}}=\mathbb{E}\left[\left\|m\odot\big(v_\theta(a^\tau,\tau\mid o,x,e)-\dot a^\tau\big)\right\|_2^2\right].
$$

这里 $e$ 为本体条件。mask 只保证损失不惩罚不存在的自由度；真正的语义对齐仍依赖动作归一化、描述、数据覆盖和后训练。预测动力学辅助目标的直觉是：动作表示不仅要“像演示”，还要编码其可预见后果。

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/samples/lingbot-vla2-fig5-stats.png' | relative_url }}" alt="LingBot VLA 2.0子任务动作统计" /><figcaption><strong>论文 Fig. 5（裁切）。</strong>子任务持续时间、频率和总时长统计揭示数据分布。它是规模/效率证据：长尾动作若没有足够样本，统一模型也可能只学到高频套路。</figcaption></figure>

## 3. 局限、硬件与 VLA Q&A

| 场景 | 论文披露 | 学习规划（个人推断） |
|---|---|---|
| 全量预训练 | 约 60k 小时、20 本体；包含大规模数据处理与模型改造 | 需多 GPU、分布式存储、视频处理和数据治理；不适合学习机全量重训 |
| 微调 | 应以官方 v2 代码、权重和版本说明为准 | 24–48GB 显存先验证单本体、小视角、离线 dataset；48–80GB/多卡适合长上下文与多相机 |
| 推理 | 输出为平台有效的 55 维子空间 | 先检查 mask、频率和归一化并做离线回放；不要把无效维置零就视为安全适配 |

**Q：阅读这篇最关键的四个字段？** `robot type`、55 维字段语义、mask、控制频率。它们决定相同数值在不同本体上代表什么。

**Q：预测动力学是否等同 WAM？** 它是为 policy 提供预测性表示的辅助目标；是否构成可用于规划的完整世界模型，要看是否显式生成/评估未来状态，而不能只看名称。

## 来源账本

- **论文事实：** [arXiv](https://arxiv.org/abs/2607.06403)，Fig. 1–5。
- **官方代码：** [lingbot-vla-v2](https://github.com/Robbyant/lingbot-vla-v2)。
- **辅助解读：** [官方技术页](https://technology.robbyant.com/lingbot-vla-v2)。
