---
layout: page
title: "OpenVLA：把连续机器人动作写进语言模型"
description: "OpenVLA 精读样板：离散动作 token、开放复现、实验解读与算力审计。"
---

> **一句话定位**：OpenVLA 是理解 VLA 的起点：它把每一维连续机器人动作量化成 token，让视觉语言模型用“下一个 token 预测”控制机械臂。它因此非常清楚地暴露了两件事：VLM 的语义迁移能力，以及离散、自回归动作表示的瓶颈。

<div class="method-flow"><span>图像 + 语言</span><b>→</b><span>Prismatic VLM</span><b>→</b><span>7 个动作 token</span><b>→</b><span>反量化 / 安全约束</span><b>→</b><span>机器人</span></div>

## 0. 阅读卡片

| 项目 | 内容 |
|---|---|
| 论文 | *OpenVLA: An Open-Source Vision-Language-Action Model*（2024） |
| 原文 / 代码 | [arXiv:2406.09246](https://arxiv.org/abs/2406.09246) · [官方仓库](https://github.com/openvla/openvla) · [本地 PDF](../../../VLA/论文/OpenVLA-2406.09246.pdf) |
| 要解决的问题 | 能否用一个开源、可微调、可部署的 7B VLA，替代封闭的大型通用机器人模型？ |
| 输入 → 输出 | 图像、语言指令、机器人状态上下文 → 7-DoF 末端动作（具体动作语义由数据集 schema 决定） |
| 最重要的实验结论 | 论文报告在 29 个真实任务上比 RT-2-X 高 **16.5 个绝对成功率百分点**；但这不是“任意机器人即插即用”的证据。 |
| 与 π0 / π0.5 的关系 | OpenVLA 代表**离散自回归动作 token**路线；π0 改用连续 flow matching，π0.5 再把高层语义预测接入动作专家。 |

**最佳技术解读**：[OpenVLA & OpenVLA-OFT 技术讲解（CSDN）](https://blog.csdn.net/m0_60827485/article/details/160480283)。适合辅助理解 OpenVLA 的动作 token 化与后续 OFT 微调思路；本文中的模型和实验事实仍以论文、官方仓库为准。

### 图谱：六类证据图

| 图类别 | 选用图 | 放置位置 |
|---|---|---|
| 模型总体框图 | Fig. 2 | §1 |
| 数据与训练配方图 | Fig. 1 | §2 |
| 输入输出 / 动作表示图 | Fig. 2（与模型总体框图合并） | §1，不重复插入 |
| 任务与环境示例图 | Fig. 3 | §4 |
| 主实验结果图 | Fig. 3（与任务环境合并） | §4，不重复插入 |
| 消融 / 效率图 | Fig. 6 | §4 |

## 1. 模型原理总览：输入、输出与模块分工

### 输入 → 输出契约

| 类别 | 论文中的内容 | 形状 / 时间语义 | 输出后处理 |
|---|---|---|---|
| 视觉输入 | 单张 RGB 观测图像 | 进入 DINOv2 与 SigLIP 两个视觉编码器 | 编码为视觉 patch 表示 |
| 语言输入 | 自然语言任务指令 | 经 Llama tokenizer 转为文本 token | 与视觉 token 一起送入 LLM |
| 动作条件 | 没有单独的连续动作状态输入作为主接口 | 模型逐维生成 7 个动作 token | 使用训练集统计量反量化 |
| 动作输出 | 7D 机器人控制动作 | 每一维是 256 个 bin 中的一个 token | 恢复为 $\Delta x,\Delta y,\Delta z,\Delta\mathrm{roll},\Delta\mathrm{pitch},\Delta\mathrm{yaw},\Delta\mathrm{grip}$（具体 schema 仍依数据集而定） |

<figure class="paper-figure">
  <img src="{{ '/assets/paper-figures/samples/openvla-fig2-architecture-crop.png' | relative_url }}" alt="裁切后的 OpenVLA Fig. 2 模型框图：双视觉编码器、投影器、Llama 2、动作反 token 化器">
  <figcaption>论文 Fig. 2（裁切）：图像经 DINOv2/SigLIP 编码并投影为视觉 token，和指令 token 一起输入 Llama 2；最后把动作 token 反 token 化为 7D 连续控制量。</figcaption>
</figure>

| 模块 | 输入 | 输出 | 职责 |
|---|---|---|---|
| DINOv2 + SigLIP | RGB 图像 | 两组视觉特征 | 同时提供空间/语义视觉表征 |
| MLP projector | 视觉特征 | 与 LLM 同维的视觉 token | 把视觉表示接到语言模型词空间 |
| Llama 2 7B | 视觉 token、指令 token、已生成动作 token | 下一个动作 token 的分布 | 以自回归 next-token prediction 建模控制 |
| action de-tokenizer | 7 个离散 token | 连续 7D 动作 | 查表/反量化并交给外部控制接口 |

**从输入到输出**：1）图像走双编码器；2）特征经 projector 对齐到 LLM token 空间；3）指令和视觉 token 组成条件上下文；4）Llama 逐个预测动作维度 token；5）de-tokenizer 用同一份归一化统计量恢复连续控制量。这个第 5 步与模型权重同样重要。

## 2. 先弄清：它到底改变了什么？

传统模仿学习常把视觉编码器、语言编码器和动作头分开训练。OpenVLA 的赌注是：互联网 VLM 已经会把“红色杯子”“放进碗里”等语义对齐，只要把机器人动作改写成词表里的新 token，VLM 的条件生成能力就可以迁移到控制上。

关键不在于“把动作离散化”本身，而在于把控制问题变成一个与语言模型接口兼容的监督问题。论文使用 Open X-Embodiment 数据混合（约 97 万轨迹），以 Llama 2、DINOv2 和 SigLIP 组成的 Prismatic VLM 为底座，得到 7B 模型。这里的泛化来自数据和视觉语言预训练；**动作 token 只是连接控制头与 VLM 的协议**。

<figure class="paper-figure">
  <img src="{{ '/assets/paper-figures/samples/openvla-fig1-overview-crop.png' | relative_url }}" alt="裁切后的 OpenVLA Fig. 1：多机器人数据、OpenVLA 和闭环控制">
  <figcaption>论文 Fig. 1（裁切）：OpenVLA 的系统级视图。左侧是约 97 万多机器人轨迹，中间是 VLM，右侧是闭环控制策略。</figcaption>
</figure>

### 它不是哪些东西

- 不是统一动作空间的魔法：不同数据集仍需在转换层明确坐标系、夹爪定义、控制频率与归一化统计量。
- 不是规划器：它主要把当前观测和指令映射为下一控制步，而非显式预测长时程未来世界状态。
- 不是对你手上机械臂可直接执行的 checkpoint：SO-ARM101 的关节/末端动作接口、相机标定和安全限幅都与论文平台不同。

## 3. 方法拆解：从连续向量到 token 序列

设时刻 $t$ 的观测为 $o_t$、文本指令为 $x$，动作向量为 $a_t\in\mathbb{R}^D$。论文对每个维度先按训练集统计量裁剪、归一化，再量化到 256 个 bin；并把 Llama 词表中末尾的 256 个低频 token 改作动作 token。动作预测因此是：

$$
p(a_t\mid o_t,x) \approx \prod_{d=1}^{D} p(z_{t,d}\mid o_t,x,z_{t,<d}), \qquad z_{t,d}\in\{0,\ldots,255\}.
$$

其中 $z_{t,d}$ 是第 $d$ 个动作维度的离散 bin；该式对应 Llama 的自回归动作 token 训练阶段。

训练损失就是动作 token 上的交叉熵：

$$
\mathcal{L}_{\text{act}}=-\sum_{d=1}^{D}\log p_\theta(z_{t,d}^{*}\mid o_t,x,z_{t,<d}).
$$

其中 $z_{t,d}^{*}$ 是示范动作量化后的监督 token，$\theta$ 是整个 VLM 的可训练参数；它只在动作 token 位置计算交叉熵。

推理时再把 bin 中心反量化回连续控制量。这种写法有三个工程后果：

1. **量化误差不可忽略**：bin 宽度由各维数据范围决定。训练集范围、裁剪阈值或动作含义一旦变化，原 checkpoint 的输出就不再可解释。
2. **维度顺序是模型的一部分**：$[\Delta x,\Delta y,\Delta z,\Delta r,\Delta p,\Delta y,gripper]$ 与另一种排列不是兼容格式；自回归还会放大前序 token 的错误。
3. **吞吐量受序列解码限制**：每次控制需顺序生成多枚 token。这是后续 FAST 和 π0 选择更高效/连续动作生成的重要背景。

## 4. 数据、训练与评估：如何读懂结果

论文的通用训练使用约 97 万 Open X-Embodiment 轨迹，并在 29 个真实机器人任务上测试。它另做了少样本下游适配：Franka 等任务每个任务收集约 10–150 条示范。这里至少要把三种数字分开：

| 数字 | 它回答的问题 | 不能推出什么 |
|---|---|---|
| 动作 token 准确率 | 模型是否拟合了离线动作标签 | 真实闭环是否稳定 |
| 成功率 | 指定基准、指定硬件和评测协议下是否完成任务 | 换相机、换控制器后仍成功 |
| 推理频率 | 某 GPU、精度、实现下的吞吐 | 真机端到端控制延迟、安全性 |

论文报告相对 RT-2-X 的 16.5 个百分点提升，且在 LIBERO 等套件中比较全参微调与参数高效微调。正确读法是：这证明“开放 7B VLA + 合理数据配方”可以是强基线；不要把跨任务平均数当成单个真实任务的保证。

<figure class="paper-figure">
  <img src="{{ '/assets/paper-figures/samples/openvla-fig3-results-crop.png' | relative_url }}" alt="裁切后的 OpenVLA Fig. 3：WidowX 任务样例和与基线的成功率比较">
  <figcaption>论文 Fig. 3（裁切，覆盖：任务与环境示例图、主实验结果图）：柱状图比较 OpenVLA、RT-2-X、Octo 与 RT-1-X 的成功率；下方任务图说明视觉、运动、物理、语义和语言泛化评测对应的操作场景。</figcaption>
</figure>

<figure class="paper-figure">
  <img src="{{ '/assets/paper-figures/samples/openvla-fig6-efficiency-crop.png' | relative_url }}" alt="裁切后的 OpenVLA Fig. 6：不同 GPU 和精度下的推理速度">
  <figcaption>论文 Fig. 6（裁切，覆盖：消融/效率图）：bf16、int8、int4 在不同 GPU 上的 actions/sec 对比。它说明量化不是只省显存，也会改变控制频率与端到端系统行为。</figcaption>
</figure>

### 辅助解读如何使用

社区文章常把 OpenVLA 说成“消费级 GPU 可用”。这个表述只在**特定推理或 LoRA 设定**下成立。本文笔记以论文和官方 README 为事实源；辅助文章只用于帮助建立直觉，不能替代对数据格式和脚本参数的核对。

## 5. 可复现性审计：从论文到你的数据，缺什么？

| 层级 | 论文 / 官方已提供 | 你仍必须自行固定的契约 |
|---|---|---|
| 模型 | 开源权重、训练与微调脚本、REST 推理示例 | checkpoint 版本、依赖版本、精度/量化路径 |
| 数据 | Open X / RLDS 生态和示例转换 | 相机帧率、时间对齐、prompt 模板、episode 切分 |
| 动作 | 逐维量化和归一化思路 | 控制模式、单位、坐标系、夹爪开闭编码、限幅 |
| 评测 | LIBERO 与真实任务协议 | 成功判据、重试数、随机化范围、人工接管规则 |

**最小离线复现建议（只学习，不接真机）**：先只取官方/公开数据的一小段，跑通“读取一条 episode → 显示图像与指令 → 验证动作归一化后再反归一化的一致性 → 离线预测”。任何一步的张量形状或单位不一致，都不应进入在线控制。

## 6. VLA 阅读 Q&A

**Q1：为什么 VLA 的泛化常常先败在动作空间？**<br>
视觉和语言可以共享语义，但控制量没有天然共同词典。不同机器人对同一个“向前”有不同关节/末端含义；因此动作 schema、归一化和频率是第一等公民。

**Q2：离散动作 token 何时合理？**<br>
当动作维度较低、控制频率可接受、需要最大化复用语言模型训练接口时很合理。高频、长动作块或精细接触操作则会更受量化和自回归延迟影响。

**Q3：离线 loss 降了为什么真机仍失败？**<br>
行为克隆只监督数据分布内动作；闭环中一个小偏差会改变后续图像分布，动作归一化/时延/相机变化都会把模型带出训练分布。

**Q4：看 VLA 实验最先问什么？**<br>
问“与基线是否同数据、同示范数、同相机、同控制频率、同评测次数”。缺少这些，成功率无法横向解释。

**Q5：OpenVLA 对 π0.5 学习最有价值的启示？**<br>
先把输入、动作和评估协议写成可审计接口；再讨论生成器是 token、flow matching 还是分层策略。方法创新无法挽救接口漂移。

## 7. 全量 / 微调 / 推理的算力与硬件审计

> 这是学习与方案评估，不是本机部署指令。你的这台电脑当前只用于阅读、数据检查和离线理解；不连接 SO-ARM101 或其他真机。

| 目标 | 论文披露 | 官方实现要求 | 显存下界估算 | 对当前 48GB GPU 的判断 |
|---|---|---|---|---|
| 从头预训练 | **64×A100，14 天，约 21,500 A100-hours，batch 2048** | 无个人级复现配方 | 数据中心级集群 | 不适用 |
| 全参下游微调 | 论文实验为 **8×A100、5–15 小时/任务** | README 推荐 8×A100 FSDP | 多卡 80GB 级较稳妥 | 不建议作为学习目标 |
| LoRA 微调 | 论文称单张 A100 约 10–15 小时 | 官方示例 batch 16 约 72GB；降低 batch 后最低约 27GB 可跑 | 27GB 是官方“可运行”下界，不等于理想配置 | 48GB 可做小批量、离线学习实验；先验证数据链路 |
| 离线推理 | bf16 权重约 15GB；RTX 4090 约 6Hz | 官方提供量化/服务示例 | 16GB 级可用 4-bit 路径，但留出图像与服务余量 | 可做离线样本推理，不接控制器 |
| 在线服务 | 论文只报告特定平台吞吐 | 需另行实现 watchdog、急停、限幅和延迟监控 | GPU 之外还需相机/控制隔离 | 当前阶段不做 |

系统层面还应预留：高速 NVMe（数据缓存与 checkpoint）、足够 RAM（数据加载），以及与机器人隔离的安全控制链。最大风险不是“显存不够”，而是把不匹配的动作统计量或延迟输出送进执行器。

## 8. 来源与证据等级

- **论文事实（一手）**：[arXiv 论文](https://arxiv.org/abs/2406.09246)，方法、数据规模、结果和训练硬件均以其为准。
- **官方实现（一手）**：[openvla/openvla](https://github.com/openvla/openvla)，LoRA/FSDP、数据转换和推理接口以 README/脚本为准。
- **本地归档**：[OpenVLA PDF](../../../VLA/论文/OpenVLA-2406.09246.pdf)。图像均为该 PDF 的原页截取，仅用于学习笔记说明。
- **个人推断**：关于 action schema、闭环分布偏移和部署风险的建议，是基于论文机制与官方接口作出的工程推论，非论文原话；未找到可替代官方材料的独立长文，因此以论文和代码为主。
