---
layout: page
title: Knowledge Insulating VLA：用离散知识保护连续控制
---

# Knowledge Insulating VLA：用离散知识保护连续控制

## 快速卡片

| 项目 | 内容 |
|---|---|
| 论文 | [Knowledge Insulating Vision-Language-Action Models: Train Fast, Perform Better, Generalize Better](https://arxiv.org/abs/2506.00150) |
| 代码 | 论文以 π0 类 Flow‑Matching VLA 为基础；请以论文作者后续公开入口为准，本文不把非官方复现当作原实现。 |
| 最佳技术解读 | [从 π0 到 π0.7 的 VLA 阅读脉络](https://fusheng-ji.github.io/blog/posts/from-pi0-to-pi07-open-pi-series/)（辅助理解“离散 token 训练主干、连续专家执行”的分工）。 |
| 核心问题 | 直接用连续动作 loss 更新 VLM，容易让语言/视觉知识被控制数据覆盖；只冻结主干又会限制适应。 |

## 1. 模型原理总览：输入、输出与模块分工

| 输入 | 模块 | 输出 |
|---|---|---|
| 图像、语言、状态 | VLM backbone | 视觉语言条件特征与离散动作 token 的预测 |
| 连续动作 chunk | Flow‑Matching action expert | 连续低层动作 |
| 离散化动作 token | Knowledge‑insulating 训练分支 | 给 VLM 的 next-token 学习信号 |

| 模块 | 分工 | 为什么需要它 |
|---|---|---|
| VLM | 理解物体、语言关系和场景知识 | 需要保持预训练知识，不应只优化电机轨迹 |
| 离散动作 token 分支 | 用 next-token 目标训练/适配 VLM | 为主干提供语言模型熟悉的监督接口 |
| 连续 action expert | 用 Flow Matching 拟合连续 chunk | 保留高频、平滑的控制表达 |
| knowledge insulation | 将“知识吸收”与“连续控制拟合”分流 | 减少直接连续 loss 对主干的干扰 |

### 图谱：六类证据图

| 图 | 覆盖类别 | 用途 |
|---|---|---|
| Fig. 1 | 模型总览；输入输出/动作表示；数据/训练 | 离散 token 分支和连续 action expert 的分工 |
| Fig. 2 | 消融/规模/效率 | 标准 VLA 训练中冻结/全训/连续目标的困难 |
| Fig. 3 | 任务/环境 | 未见环境与多本体评测设定 |
| Fig. 4 | 主结果 | 代表任务上的方法对比 |

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/samples/ki-fig1-method.png' | relative_url }}" alt="Knowledge Insulating VLA的离散token和连续动作专家框图" /><figcaption><strong>论文 Fig. 1（裁切）。</strong>方法把 VLM 的知识学习信号转为离散动作 token 的 next-token 预测，同时保留 Flow‑Matching expert 生成连续动作；同一图覆盖结构、I/O 和训练分工。</figcaption></figure>

### 4 步理解法

1. 用观测与指令构建 VLM 条件表示。
2. 将演示动作转换为离散 token，令主干通过 next-token 目标学习动作语义。
3. 由连续 action expert 在同一条件下学习速度场并生成连续 chunk。
4. 将两者在训练职责上隔离：主干重点保留知识，expert 重点拟合精细控制。

## 2. 方法动机与公式

论文的判断是：连续控制 loss 的信号密度高但语义弱，若它直接大规模更新 VLM，可能造成知识遗忘；而“完全冻结 VLM”虽然稳定，却把视觉语言适应能力锁死。离散 token 让主干继续接受类似语言建模的监督，连续 expert 则处理精确轨迹。

离散分支可写成 next-token 负对数似然：

$$
\mathcal{L}_{\mathrm{disc}}=-\sum_j\log p_\theta(z_j\mid z_{<j},o,x).
$$

连续专家仍使用 Flow‑Matching 形式：

$$
\mathcal{L}_{\mathrm{FM}}=\mathbb{E}\left[\left\|v_\phi(a^\tau,\tau\mid o,x)-\big(a^1-a^0\big)\right\|_2^2\right].
$$

$z_j$ 是离散动作 token，$a^0$ 是演示动作 chunk，$a^1$ 是噪声，$v_\phi$ 是 expert 的速度场。两式对应不同模块，不能误读为“把连续动作完全离散化后执行”。

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/samples/ki-fig2-problem.png' | relative_url }}" alt="Knowledge Insulating VLA对标准训练方案问题的示意" /><figcaption><strong>论文 Fig. 2（裁切）。</strong>作者用标准配方的训练/性能现象说明：仅冻结或直接连续微调都存在问题。它是诊断证据，结论仍需结合对应任务和训练预算阅读。</figcaption></figure>

## 3. 任务、结果与可迁移结论

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/samples/ki-fig3-tasks.png' | relative_url }}" alt="Knowledge Insulating VLA的评测任务环境" /><figcaption><strong>论文 Fig. 3（裁切）。</strong>论文将完全未见环境、多本体/多任务设为评测；阅读主结果前，先确认训练数据中是否出现过相似对象、背景和动作原语。</figcaption></figure>

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/samples/ki-fig4-results.png' | relative_url }}" alt="Knowledge Insulating VLA的代表性结果比较" /><figcaption><strong>论文 Fig. 4（裁切）。</strong>代表任务比较用于验证知识隔离设计的收益。应按同一动作空间、同一数据量和同一试验协议比较，而不是跨图挑选最高分。</figcaption></figure>

可迁移的设计语言是：把“主干要学什么知识”与“执行头要拟合什么连续信号”显式拆开；它适用于分析 VLA 后训练，但不是保证开放世界泛化的充分条件。

## 4. 局限、算力与硬件审计

| 场景 | 论文披露 | 学习规划（个人推断） |
|---|---|---|
| 全量训练 | 论文关注训练配方，不提供消费级单机的完整全量训练预算 | 需要多 GPU、原始 VLM 权重、混合机器人数据和严格动作标注；不建议学习机从零复刻 |
| 微调 | 方法本质上多了离散 token 监督和连续 expert 的联合训练 | 先在 24–48GB 显存上做小数据 LoRA/adapter 与离线 loss 验证；48–80GB 可降低长序列、图像与 batch 的折衷 |
| 推理 | 连续 expert 负责输出 action chunk | 需单独测试采样步数与端到端时延；离线回放优先，不直接接入真机 |

风险包括：tokenizer/归一化不一致、主干与 expert 条件错位、冻结策略误配，以及把 OOD benchmark 等同于真实安全。

## 5. VLA 阅读 Q&A

**Q：知识隔离到底隔离什么？** 隔离的是主干接收的训练信号与连续控制拟合的职责，不是让模型“忘记动作”。离散 token 让主干保持语言式预测接口，expert 仍承担连续动作。

**Q：复现优先检查哪项？** 先检查离散 token 如何从动作生成、它与 action expert 是否共享条件、哪个模块被更新、动作归一化与 horizon 是否一致。

## 来源账本

- **论文事实：** [arXiv 论文](https://arxiv.org/abs/2506.00150)，重点 Fig. 1–4 与方法章节。
- **辅助解读：** [π0 系列阅读脉络](https://fusheng-ji.github.io/blog/posts/from-pi0-to-pi07-open-pi-series/)；用于概念对照，具体事实回到论文。
- **个人推断：** 硬件规划仅用于学习与离线实验准备。
