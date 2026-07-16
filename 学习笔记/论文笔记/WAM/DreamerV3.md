---
layout: page
title: DreamerV3：在潜空间中想象并学习控制
---

# DreamerV3：在潜空间中想象并学习控制

## 快速卡片

| 项目 | 内容 |
|---|---|
| 论文 | [Mastering Diverse Domains through World Models](https://arxiv.org/abs/2301.04104) |
| 官方实现 | [danijar/dreamerv3](https://github.com/danijar/dreamerv3) |
| 最佳技术解读 | [Danijar Hafner 项目页](https://danijar.com/project/dreamerv3/)，与官方代码配置一起用于理解世界模型、imagination 与跨域设置。 |
| 核心结论 | 固定一套超参数跨连续控制、Atari、ProcGen、DMLab、Minecraft 等视觉域；关键是学 world model 后在 latent 中训练 actor/critic。 |

## 1. 模型原理总览：输入、输出与模块分工

| 输入 | 中间状态 | 输出 |
|---|---|---|
| 图像观测 $o_t$、动作 $a_t$ | RSSM 的确定性状态 $h_t$ 与离散随机状态 $z_t$ | 奖励/继续概率预测、价值、下一动作 |

| 模块 | 做什么 |
|---|---|
| Encoder / decoder | 图像与 latent 双向映射，用重建约束表征 |
| RSSM dynamics | 由过去状态和动作预测 latent prior，形成“想象世界” |
| Representation posterior | 看到真实观测时修正 latent，训练时与 prior 对齐 |
| Reward/continue heads | 预测奖励与 episode 是否继续，支持 imagined return |
| Actor / critic | 在 latent rollout 上优化策略与价值，不需每个梯度步真实交互 |

### 图谱：六类证据图

| 图 | 覆盖类别 | 用途 |
|---|---|---|
| Fig. 3 | 模型总览；输入输出/动作表示；数据/训练 | RSSM、encoder、decoder、actor/critic 数据流 |
| Fig. 2 | 任务/环境 | 跨机器人、游戏、程序生成和 Minecraft 的评测域 |
| Fig. 4 | 主结果；任务/环境 | 多步视频预测定性证据 |
| Fig. 6 | 消融/规模/效率 | robustness 技术与模型规模的消融 |

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/samples/dreamer-fig3-rssm.png' | relative_url }}" alt="DreamerV3 RSSM世界模型和actor critic框图" /><figcaption><strong>论文 Fig. 3（裁切）。</strong>真实观测用于 posterior 校正，dynamics 负责 imagined rollout，actor/critic 在潜空间优化；这张图覆盖模型、I/O 与训练分工。</figcaption></figure>

### 5 步从真实数据到想象控制

1. 编码器把 $o_t$ 变为观测特征。
2. RSSM posterior 得到 $z_t$，并与仅依赖过去/动作的 prior 对齐。
3. 训练 decoder、reward 和 continue head，使 latent 有可预测的世界语义。
4. 从 latent 状态 rollout 多步 imagined trajectory。
5. actor 最大化 imagined return，critic 估值；再用真实环境新数据纠正模型偏差。

## 2. 目标函数：世界模型如何变成策略学习信号

RSSM 的常见训练形式可概括为：

$$
\mathcal{L}_{\mathrm{wm}}=\mathcal{L}_{\mathrm{rec}}(o_t\mid h_t,z_t)+\mathcal{L}_{\mathrm{rew}}(r_t\mid h_t,z_t)+\mathcal{L}_{\mathrm{cont}}+\beta\,D_{\mathrm{KL}}\big(q(z_t\mid h_t,o_t)\|p(z_t\mid h_t)\big).
$$

重建、奖励、继续概率与 KL 项共同让 latent 既“看得像”又“与决策有关”。世界模型评价不能止于像素清晰度：如果 reward/continue 或动作后果预测错，actor 会在想象中利用模型漏洞。

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/samples/dreamer-fig2-domains.png' | relative_url }}" alt="DreamerV3跨视觉域任务" /><figcaption><strong>论文 Fig. 2（裁切）。</strong>任务环境从机器人运动到 Atari、DMLab、Minecraft；它说明 DreamerV3 的“泛化”首先是固定超参数跨域，而非单一机器人零样本承诺。</figcaption></figure>

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/samples/dreamer-fig4-prediction.png' | relative_url }}" alt="DreamerV3多步视频预测" /><figcaption><strong>论文 Fig. 4（裁切）。</strong>多步预测提供直观证据，但应结合 reward/任务成功检验；视觉合理并不保证接触或长期规划正确。</figcaption></figure>

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/samples/dreamer-fig6-ablation.png' | relative_url }}" alt="DreamerV3鲁棒性和规模消融" /><figcaption><strong>论文 Fig. 6（裁切）。</strong>消融将归一化、KL 平衡、symlog 等稳健化设计与规模表现关联起来。复现时不要只复制网络层数而漏掉这些训练细节。</figcaption></figure>

## 3. 算力/硬件审计与 WAM Q&A

| 场景 | 论文事实 | 学习规划（个人推断） |
|---|---|---|
| 全量跨域训练 | DreamerV3 给出统一配置跨多类环境的研究结果 | 主要成本来自环境交互、图像 replay 和多次 world-model 更新；多 GPU/并行环境有利，不是简单大显存即可 |
| 单环境复现 | 官方代码可在较小视觉任务开始 | 12–24GB GPU 可做小型环境/缩小模型的学习复现；重视 replay 存储、环境吞吐和随机种子 |
| 机器人迁移 | 论文含机器人域但不是面向具体真机部署指南 | 先仿真与离线日志；模型误差、时延和安全约束不能由世界模型自动保证 |

**Q：WAM 最应看什么？** 观察模型误差会不会在 imagined horizon 上累积、actor 是否 exploit 模型、真实环境回报是否跟随 imagined 改善。

## 来源账本

- **论文事实：** [arXiv](https://arxiv.org/abs/2301.04104)，Fig. 2/3/4/6。
- **官方代码：** [dreamerv3](https://github.com/danijar/dreamerv3)。
- **辅助解读：** [项目页](https://danijar.com/project/dreamerv3/)。
