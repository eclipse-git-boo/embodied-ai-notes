---
layout: page
title: "π0：用 Flow Matching 生成连续动作块"
description: "π0 精读样板：VLM 初始化、动作专家、flow matching 与复现边界。"
---

> **一句话定位**：π0 不再把动作逐维当作语言 token 顺序输出，而是在预训练 VLM 上增加一个约 300M 参数的动作专家，用 flow matching 并行生成连续动作块。这是从 OpenVLA 的“token 化控制”迈向连续、高频控制的重要转折。

<div class="method-flow"><span>图像 / 指令 / 状态</span><b>→</b><span>PaliGemma VLM</span><b>→</b><span>动作专家</span><b>→</b><span>Flow matching 去噪</span><b>→</b><span>H=50 连续动作</span></div>

## 1. 阅读卡片

| 项目 | 内容 |
|---|---|
| 论文 | *π0: A Vision-Language-Action Flow Model for General Robot Control*（2024） |
| 原文 / 代码 | [论文主页](https://www.physicalintelligence.company/research/π0) · [PDF](https://www.physicalintelligence.company/download/pi0.pdf) · [openpi 官方仓库](https://github.com/Physical-Intelligence/openpi) · [本地 PDF](../../../VLA/论文/pi0.pdf) |
| 核心命题 | 用预训练 VLM 的语言/视觉能力作条件编码，用连续生成模型处理高维、快速且跨形态的动作。 |
| 结构 | PaliGemma 风格 VLM（Gemma 2B 语言底座）+ 独立参数的动作 expert（约 300M）。 |
| 动作输出 | 论文使用动作 horizon $H=50$；动作 expert 在双向注意力下并行预测连续动作 token。 |
| 对 π0.5 的关系 | π0 给出“VLM + 连续动作 expert”的骨架；π0.5 在其上加入多阶段训练和高层语义/子任务预测。 |

## 2. 为什么 OpenVLA 的思路还不够？

离散 token 的优点是能直接套用自回归语言模型；但高自由度控制会出现两个问题。第一，连续动作被量化，精细接触和速度控制受 bin 精度限制；第二，若按维度或时间自回归生成，延迟与误差会沿序列累积。π0 的判断是：语言 token 与连续控制 token 应使用不同生成机制。

它将问题写成条件动作序列建模：给定观测 $o_t$、语言 $ell_t$、本体状态 $s_t$，直接生成未来动作块 $a_{t:t+H}$。VLM 仍负责“看懂”和“听懂”，动作 expert 则负责“平滑、快速地执行”。这不是把 VLM 扔掉，而是将视觉语言与低层控制的归纳偏好分开。

<figure class="paper-figure">
  <img src="{{ '/assets/paper-figures/samples/pi0-fig1.png' | relative_url }}" alt="π0 论文首页和模型概览">
  <figcaption>论文 Fig. 1 所在页：π0 将预训练 VLM 与连续动作生成专家结合，面向多机器人形态与多任务控制。图像来自论文 PDF 原页。</figcaption>
</figure>

## 3. 方法：条件 Flow Matching 在做什么？

先从高斯噪声动作块 $a^0$ 与示范动作块 $a^1$ 构造插值路径。简化写作：

\[
a^\tau=(1-\tau)a^0+\tau a^1,\quad \tau\sim p(\tau),
\]

并训练向量场 $v_\theta$ 预测把噪声推向数据的速度：

\[
\mathcal{L}_{\mathrm{FM}}=\mathbb{E}\left[\left\|v_\theta(a^\tau,\tau,o_t,\ell_t,s_t)-(a^1-a^0)\right\|_2^2\right].
\]

推理时，从噪声动作块出发，用数值积分沿 $v_\theta$ 走到 $\tau=1$，一次得到整段连续动作。论文的实现将动作 token 投影进 transformer；动作 token 使用双向 attention，可相互协调，而文本 token 仍遵循生成顺序。这是“action expert”存在的技术原因：它给动作提供了不同于文本的注意力与损失。

<figure class="paper-figure">
  <img src="{{ '/assets/paper-figures/samples/pi0-fig2.png' | relative_url }}" alt="π0 跨机器人数据和任务示例">
  <figcaption>论文 Fig. 2 所在页：训练/评测覆盖不同机器人形态和任务。跨形态不等于零转换成本；每个平台的状态、动作与时间同步仍需显式定义。</figcaption>
</figure>

### 三个容易误读的点

| 容易误读 | 更准确的理解 |
|---|---|
| “flow matching 自动更安全” | 它改善连续动作生成方式，不提供碰撞检测、急停或工作空间约束。 |
| “一次生成 50 步就不用闭环” | 动作块仍应在闭环中重规划；执行多少步再刷新，是控制频率/时延共同决定的工程参数。 |
| “跨 embodiment 等于直接迁移” | 论文展示跨平台数据带来的能力，但动作接口、相机、状态字段和标定仍需转换。 |

## 4. 训练数据与实验该怎样读

论文把不同机械臂、双臂、移动操作等数据放进同一训练混合中，并在 68 个任务、7 类机器人设置上报告结果。它特别强调高频/灵巧任务：论文附录报告在 RTX 4090 上进行推理的设置，并针对移动机器人以固定间隔重新推理、执行部分 action chunk。

这里有一个重要实验习惯：不要只看“总成功率”。应分别记录任务是否需要高频控制、是否使用本体状态、动作维度是否相同、评测是否允许重试，以及模型每次生成后实际执行多少个动作。后四项一变，flow model 与自回归模型的比较就可能不再公平。

<figure class="paper-figure">
  <img src="{{ '/assets/paper-figures/samples/pi0-results.png' | relative_url }}" alt="π0 实验结果页">
  <figcaption>论文实验结果页：包含不同任务/基线的表现。阅读数字前先核对任务难度、训练数据量、动作频率和成功判据，而不是只摘取最高分。</figcaption>
</figure>

### 来自辅助解读的有效直觉（已重新表述）

一篇 [CSDN 技术解读](https://blog.csdn.net/v_JULY_v/article/details/143472442) 将 π0 概括为“VLM 管条件理解，动作 expert 管连续生成”，这个分工有助于初读；该文也在后续修订中更正过架构细节。模型规模、数据小时数、4090 延迟等所有具体数字仍以论文 PDF 和官方实现为准。不能把二手文章中的经验配置直接当作复现实验配置。

## 5. 复现路线：先验收接口，再谈训练

openpi 是 π0/π0.5 的官方开源实现入口，并提供 JAX/PyTorch 路径、LeRobot 数据转换、归一化统计量和 policy 服务示例。官方也明确指出，开源项目主要面向**微调**，并不公开完整预训练数据与训练规模；因此“从零复现论文能力”并不是可验证目标。

| 阶段 | 离线验收问题 | 通过标准 |
|---|---|---|
| 数据 | 每条 observation 是否与 action 时间对齐？ | 随机抽样可追溯到同一 episode，字段单位清楚 |
| 归一化 | train / eval 是否使用同一份 norm stats？ | 归一化后反变换误差接近数值精度 |
| 模型 | 动作 chunk 的维度和 horizon 是否匹配？ | 模型输出形状为 `[H, action_dim]` 且无静默 padding 错位 |
| 评估 | 是否只在已录制数据上比较？ | 先报告离线指标与可视化，不把它等同真机成功率 |

对 π0 学习最有价值的不是立刻训练，而是弄清“**动作块何时重新生成、何时执行、由谁限幅**”。这三个问题在任何连续动作 VLA 上都比模型名称更直接地决定风险。

## 6. VLA 阅读 Q&A

**Q1：π0 为什么要保留 VLM，而不是只训练 diffusion/flow policy？**<br>
VLM 提供语言和视觉的预训练语义；flow expert 提供适配连续动作联合分布的生成偏好。二者解决的不是同一件事。

**Q2：flow matching 与 diffusion 的关系是什么？**<br>
二者都把生成看成从噪声到数据的连续变换。这里用 flow matching 学一个速度场，再用积分得到动作；实际效果还依赖时间采样、积分步数和网络结构，不能只凭名称判断优劣。

**Q3：为什么动作 expert 要双向注意力？**<br>
同一动作块内的未来各步相互约束。允许它们相互看见，有利于并行协调整段连续轨迹；这与文本的自回归因果生成需求不同。

**Q4：action chunk 越长越好吗？**<br>
不一定。长 chunk 降低重规划频率，却会增加观察过时后的漂移。应把 chunk 长度、执行子步数和相机端到端时延一起调。

**Q5：π0 对 π0.5 最重要的遗产是什么？**<br>
是“离散语义 token 与连续动作 token 可由同一 VLM 条件、不同 expert 分工”的架构。π0.5 的高层子任务预测正建立在这一分工之上。

## 7. 全量 / 微调 / 推理的算力与硬件审计

> 当前用途仅限学习笔记、代码阅读和离线数据检查。以下不是对 SO-ARM101/NERO 的部署授权，也不应直接把模型输出接进控制器。

| 目标 | 论文披露 | 官方实现要求 | 显存下界估算 | 当前 48GB GPU 的学习定位 |
|---|---|---|---|---|
| 全量预训练 | **未知**：论文未公开完整可复现 GPU 配方与全部数据 | openpi 不提供预训练复刻流程 | 数据中心级多卡训练，无法由论文给出可靠单卡数字 | 不作为目标 |
| 全参微调 | **未知**：论文未给出统一显存表 | PyTorch 路径当前有单 GPU 限制，且不支持 FSDP / mixed precision / LoRA（以官方 README 为准） | 需按实际 checkpoint、序列长度和优化器实测 | 不建议在当前阶段尝试 |
| 参数高效微调 | **未知**：论文未发布官方 LoRA 配方 | 官方仓库的可用训练配置优先于博客命令 | 48GB 可能只够小 batch/短序列的探索，不能据此承诺可训 | 仅离线阅读与小样本 sanity check |
| 离线推理 | 论文附录报告 RTX 4090 推理实验，但具体延迟依赖平台 | openpi 提供 policy 服务与样例配置 | 取决于 checkpoint、相机数、积分步数；应实测 | 可做录制数据推理和延迟测量 |
| 在线服务 | **未知**：不等价于论文的机器人实验系统 | 需独立安全层、watchdog、急停和限幅 | GPU 只是其中一环 | 当前不做 |

除 GPU 外，完整研究环境还需 NVMe 数据盘、足够 RAM、稳定相机时间戳和独立的控制安全层。π0 的特有风险是：动作块、积分步数和执行频率形成耦合，单独“加快模型”不一定使系统更安全。

## 8. 来源与证据等级

- **论文（一手）**：[π0 PDF](https://www.physicalintelligence.company/download/pi0.pdf)：模型、$H=50$、动作 expert、实验和 RTX 4090 相关描述。
- **官方实现（一手）**：[Physical-Intelligence/openpi](https://github.com/Physical-Intelligence/openpi)：数据格式、配置、训练能力边界与服务接口。
- **辅助解读（二手）**：用于解释直觉，不迁移原文、图表或未经论文核对的参数。
- **本地归档**：[π0 PDF](../../../VLA/论文/pi0.pdf)，本文三幅图均截自该原始 PDF 页面。
