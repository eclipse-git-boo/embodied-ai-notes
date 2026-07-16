---
layout: page
title: FAST / π0-FAST：把高频动作压缩为可预测 token
---

# FAST / π0‑FAST：把高频动作压缩为可预测 token

## 快速卡片

| 项目 | 内容 |
|---|---|
| 论文 | [FAST: Efficient Action Tokenization for Vision-Language-Action Models](https://arxiv.org/abs/2501.09747) |
| 官方代码/资源 | [Physical Intelligence FAST 项目页](https://www.pi.website/research/fast)；[openpi](https://github.com/Physical-Intelligence/openpi) |
| 最佳技术解读 | [Hugging Face：π0 and π0‑FAST](https://huggingface.co/blog/pi0)，解释了 action token、注意力接口及 FAST 的 DCT→BPE 流程。 |
| 核心结论 | 先压缩连续动作 chunk，再自回归预测 token；在论文设置中，π0‑FAST 可匹配 Flow‑Matching VLA 的表现，训练最多快 5 倍。 |

## 1. 模型原理总览：输入、输出与模块分工

### 输入—输出契约

| 输入 | 经过什么 | 输出 |
|---|---|---|
| 图像、语言指令、可选机器人状态 | VLA 主干作为条件上下文 | 压缩后的离散 action token 序列 |
| 连续动作 chunk $a\in\mathbb{R}^{T\times D}$ | 归一化、DCT、量化、展平、BPE | token 序列 $z$ |
| 自回归 token $\hat z$ | BPE 反解、反量化、IDCT | 连续动作 $\hat a$ |

| 模块 | 职责 |
|---|---|
| 分位数归一化 | 将各维动作对齐到约 $[-1,1]$，降低本体量纲差异与异常值影响 |
| DCT | 把时间域的相关轨迹移到频域，集中低频信息 |
| scale-and-round | 量化频率系数，让大量小系数归零，形成稀疏结构 |
| BPE tokenizer | 合并跨维/跨时间常见模式，缩短 token 序列 |
| 自回归 VLA | 在视觉语言条件下预测压缩 token；不是直接预测每一维每一帧的 bin |

### 图谱：六类证据图

| 图 | 覆盖类别 | 用途 |
|---|---|---|
| Fig. 2 | 模型总览；输入输出/动作表示；主结果 | FAST 接到 VLA 后为何能处理高频控制，并与 binning 比较 |
| Fig. 4 | 数据/训练；输入输出/动作表示 | DCT、量化、展平、BPE 的完整变换链 |
| Fig. 5 | 任务/环境 | 论文评测的操作环境和不同本体任务 |
| Fig. 1 | 主结果；消融/规模/效率 | π0‑FAST 与 π0 的训练速度/表现关系 |

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/samples/fast-fig2-tokenizer.png' | relative_url }}" alt="FAST连接到VLA的动作token化示意和频率比较" /><figcaption><strong>论文 Fig. 2（裁切）。</strong>左侧显示高频动作经 FAST 变成自回归 token；右侧比较不同动作 token 化方法在各控制频率下的得分。这一张图同时承担模型接口、动作表示和核心结果证据。</figcaption></figure>

### 从观测到动作的 5 步

1. 收集一段长度 $T$、维度 $D$ 的连续动作，并按训练集统计量归一化。
2. 每个动作维度沿时间轴做 DCT，得到频率系数矩阵。
3. 量化并优先保留低频、非零系数；展平后由 BPE 编码成 $z$。
4. 视觉语言主干在图像/指令条件下自回归产生 $\hat z$。
5. 逆 BPE、反量化与 IDCT 还原 $\hat a$。真实系统仍需独立的限幅、同步和安全层；本笔记不进行真机控制。

## 2. 方法：为什么 DCT + BPE 能替代逐帧分桶

朴素 VLA 常把每一维、每一时间步独立离散化。高频动作的相邻帧强相关，结果是 token 很多、局部复制就能降低 loss，却不一定学到完整运动。FAST 将一个 chunk 当作时间序列压缩：低频项优先保存平滑的主要运动，高频项只在必要时占用 token；BPE 再复用常见符号模式。

对一维归一化动作 $a_0,\ldots,a_{T-1}$，DCT 可写为：

$$
c_k=\sum_{t=0}^{T-1}a_t\cos\left(\frac{\pi}{T}\left(t+\frac12\right)k\right),\quad k=0,\ldots,T-1.
$$

$c_k$ 是第 $k$ 个频率系数。FAST 对 $c_k$ 缩放并取整，然后 BPE 编码；解码端执行可逆的反向步骤。关键不在“DCT 天然最好”，而在于它把动作相关性显式变成可压缩结构。

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/samples/fast-fig4-pipeline.png' | relative_url }}" alt="FAST动作token化的DCT量化展平和BPE流程" /><figcaption><strong>论文 Fig. 4（裁切）。</strong>从归一化 action chunk 到 DCT、量化、按低频优先展平和 BPE 的流程。复现时必须把归一化统计、chunk 长度、缩放系数和 tokenizer 版本一起保存。</figcaption></figure>

## 3. 数据、任务与证据

论文提出 FAST+：在约 100 万条真实机器人动作轨迹上训练的通用 tokenizer；并将 FAST 与 π0 结合，在约 10k 小时机器人数据上训练 π0‑FAST。该工作专门针对高频、灵巧操作，而不是只在低频桌面基准验证 token 长度。

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/samples/fast-fig5-tasks.png' | relative_url }}" alt="FAST评测中的机器人操作环境" /><figcaption><strong>论文 Fig. 5（裁切）。</strong>多种机械臂、移动操作与灵巧任务构成评测环境；任务多样性不能替代对每台机器人动作定义和控制频率的核对。</figcaption></figure>

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/samples/fast-fig1-overview.png' | relative_url }}" alt="FAST论文首页的效果与效率概览" /><figcaption><strong>论文 Fig. 1（裁切）。</strong>论文以 π0‑FAST 展示在复杂灵巧任务上的可行性，并报告训练最多 5× 加速。注意：自回归解码不必然比 Flow‑Matching 推理更快，作者也明确讨论这一权衡。</figcaption></figure>

## 4. 复现检查单与局限

- **动作统计必须同源。** 训练、评测、解码三处的分位数归一化不一致，IDCT 后即使图像正常也会产生错误幅值。
- **tokenizer 不等于 policy。** FAST+ 可作为通用接口，但 BPE 词表、缩放系数、chunk 和控制频率仍可能需要按数据审计。
- **效率要分开测。** token 更短通常降低训练序列成本；自回归生成的时延却与 token 数、缓存和硬件有关，应单独计时。
- **不把离线成功率当成安全。** 限位、碰撞、相机时延和急停不属于 tokenizer 的能力边界。

## 5. 全量训练、微调与推理：算力/硬件审计

| 场景 | 论文披露 | 学习规划（个人推断） |
|---|---|---|
| FAST tokenizer 训练 | FAST+ 用约 100 万条真实动作轨迹；未披露可直接复刻的完整 GPU-hours | 先在离线小数据上校验 encode→decode MSE、token 长度和边界动作；CPU 即可做前处理，GPU 取决于 VLA 训练而非 DCT 本身 |
| π0‑FAST 全量训练 | 论文展示 10k 小时规模与最多 5× 训练加速；没有公布个人可复现的总算力下限 | 不适合学习机全量重训；需要多 GPU、分布式训练、数据吞吐和严格的数据许可 |
| 微调/离线推理 | 官方实现与版本应优先于本文估计 | 24–48GB 显存可作为小批 LoRA/评测的规划起点；48–80GB 更适合图像 VLA 与较长序列，先做离线回放 |

## 6. VLA 阅读 Q&A

**Q：这篇应重点核对什么？** 动作 chunk 的 $T,D$、归一化、DCT 顺序、量化尺度、BPE 词表、控制频率和逆变换。它们共同定义“一个 token 到底对应什么动作”。

**Q：FAST 与 π0 的 Flow Matching 谁更适合？** FAST 适合研究自回归 VLA 如何获得高频动作接口；π0 的连续专家通常有不同的推理时延特性。不能只凭训练吞吐选择。

## 来源账本

- **论文事实：** [arXiv](https://arxiv.org/abs/2501.09747)，重点 Fig. 1/2/4/5 与实验章节。
- **官方代码/资源：** [FAST 项目页](https://www.pi.website/research/fast)、[openpi](https://github.com/Physical-Intelligence/openpi)。
- **辅助解读：** [Hugging Face 技术博客](https://huggingface.co/blog/pi0)，用于理解 π0/π0‑FAST 的实现语境；具体数字以论文为准。
- **个人推断：** 硬件规划为离线学习建议，不是官方配置承诺。
