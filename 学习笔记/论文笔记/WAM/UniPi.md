---
layout: page
title: UniPi：以文本条件视频生成作为通用策略
---

# UniPi：以文本条件视频生成作为通用策略

## 快速卡片

| 项目 | 内容 |
|---|---|
| 论文 | [Text-Conditional Video Generation as Universal Policies](https://arxiv.org/abs/2302.00111) |
| 官方代码 | [UniPi 项目仓库](https://github.com/ChengyiFei/UniPi) |
| 最佳技术解读 | [UniPi 官方项目/README](https://github.com/ChengyiFei/UniPi)；当前最接近方法和配置的一手讲解。 |
| 核心问题 | 不直接输出动作，而是先生成满足语言目标的视频计划，再用逆动力学模型从相邻帧提取动作。 |

## 1. 模型原理总览：输入、输出与模块分工

| 输入 | 中间计划 | 最终输出 |
|---|---|---|
| 当前图像 $o_t$、文本目标 $x$ | 文本条件视频 $hat o_{t+1:t+H}$ | inverse dynamics 从 $(o_t,\hat o_{t+1})$ 生成动作 $a_t$ |

| 模块 | 做什么 |
|---|---|
| Video diffusion | 将目标文本与当前图像扩展成未来帧序列，是计划器 |
| 分层采样/约束 | 在测试时通过目标、约束或中间帧引导计划 |
| Inverse dynamics model | 将视频相邻帧的变化译为低层动作 |
| 视觉计划验证 | 人可检查视频计划是否向目标推进，是可解释性来源之一 |

### 图谱：六类证据图

| 图 | 覆盖类别 | 用途 |
|---|---|---|
| Fig. 1 | 模型总览；数据/训练；输入输出/动作表示 | 文本→视频计划→逆动力学动作的完整链 |
| Fig. 2 | 输入输出/动作表示 | 从观测与文本逐帧生成计划的过程 |
| Fig. 3 | 主结果；任务/环境 | 未见组合语言目标的生成证据 |
| Fig. 4 | 消融/规模/效率；任务/环境 | 视频计划如何转换为可执行动作 |

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/samples/unipi-fig1-overview.png' | relative_url }}" alt="UniPi文本条件视频生成到动作策略" /><figcaption><strong>论文 Fig. 1（裁切）。</strong>UniPi 用文本条件视频生成替代直接 policy，再通过逆动力学执行计划；该图覆盖模型、训练语境与 I/O。</figcaption></figure>

### 从指令到动作的 4 步

1. 给当前视觉观察和语言目标编码条件。
2. 视频扩散模型产生一段向目标推进的未来帧计划。
3. inverse dynamics 将当前帧与下一计划帧的差异变为控制动作。
4. 执行后观察新图像并重规划；计划失败可在视频层被发现，但不表示动力学已经准确。

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/samples/unipi-fig2-process.png' | relative_url }}" alt="UniPi视频条件生成过程" /><figcaption><strong>论文 Fig. 2（裁切）。</strong>视频生成过程把语言目标转成时序视觉计划。阅读时要检查生成长度、replanning 间隔以及 inverse dynamics 的训练分布。</figcaption></figure>

## 2. 方法公式与直觉

视频生成可抽象为条件扩散目标：

$$
\mathcal{L}_{\mathrm{video}}=\mathbb{E}_{\epsilon,\tau}\left[\|\epsilon-\epsilon_\theta(z^\tau,\tau\mid o_t,x)\|_2^2\right],\qquad a_t=f_{\mathrm{IDM}}(o_t,\hat o_{t+1}).
$$

第一式学习计划视频，第二式表示动作提取接口。UniPi 的关键假设是“任务相关动作可从视觉状态改变推断”；这在遮挡、接触力、相机运动或多个等价动作时会变弱。

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/samples/unipi-fig3-generalization.png' | relative_url }}" alt="UniPi组合泛化视频计划" /><figcaption><strong>论文 Fig. 3（裁切）。</strong>组合语言目标下的视频计划是主结果证据。须区分“生成画面含目标”与“逆动力学能稳定执行该计划”。</figcaption></figure>

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/samples/unipi-fig4-action.png' | relative_url }}" alt="UniPi从视频计划到动作执行" /><figcaption><strong>论文 Fig. 4（裁切）。</strong>动作执行链表明 UniPi 的低层控制依赖 IDM。它也是效率/应用证据：每次重规划需要视频采样，实际延迟不可忽略。</figcaption></figure>

## 3. 局限、硬件与 WAM Q&A

| 场景 | 论文事实 | 学习规划（个人推断） |
|---|---|---|
| 全量训练 | 文本条件视频模型需要大量视频数据/扩散训练 | 多 GPU、视频存储和较长训练周期；不建议个人全量训练 |
| 微调 | 需同时关注视频模型与 IDM 的数据分布 | 24GB 可做小分辨率短计划离线实验；48GB 以上更适合稳定视频扩散微调 |
| 推理 | 每轮规划要采样视频再走 IDM | 先离线评估计划一致性/动作误差与 p95 延迟；勿将生成计划直接接入真机 |

**Q：UniPi 与直接 VLA 的区别？** 它把行动先表述为“未来会看到什么”，再反推动作；优势是计划可见，代价是视频采样和 IDM 误差成为新瓶颈。

## 来源账本

- **论文事实：** [arXiv](https://arxiv.org/abs/2302.00111)，Fig. 1–4。
- **官方代码：** [UniPi](https://github.com/ChengyiFei/UniPi)。
- **辅助解读：** 官方 README；未找到覆盖方法细节且可核验的更完整独立技术长文。
