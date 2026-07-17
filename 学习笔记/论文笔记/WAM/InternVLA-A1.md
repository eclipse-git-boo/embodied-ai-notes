---
layout: page
title: "InternVLA-A1：以未来视觉连接理解与动作"
description: "WAM 视角精读：三专家 MoT、混合数据与动态操控。"
paper_domain: WAM
---

> **一句话结论**：InternVLA-A1 是操控 VLA，但以“预测动作后的未来视觉”作为中间能力；理解专家提供语义，生成专家学习动力学，动作专家用 Flow Matching 输出连续动作，目标是让策略对动态场景更有预见性。

## 0. 快读卡片

| 项目 | 内容 |
|---|---|
| 论文 | [InternVLA-A1: Unifying Understanding, Generation and Action for Robotic Manipulation](https://arxiv.org/abs/2601.02456)（2026） |
| 官方资源 | [InternVLA-A-series](https://github.com/InternRobotics/InternVLA-A-series) · [InternData-A1](https://huggingface.co/datasets/InternRobotics/InternData-A1) |
| 最佳技术解读 | [论文原文与项目代码](https://github.com/InternRobotics/InternVLA-A-series)：截至笔记撰写时未找到比官方材料更完整且可核验的独立长文；以下事实以论文为准。 |
| 定位 | **主范式是 VLA；本页按 WAM 阅读**，因为未来视觉生成被用于动作条件表征。 |

### 图谱：六类证据图

| 类别 | 证据 | 放置位置 |
|---|---|---|
| 模型总体、I/O、动作 | Fig. 2 | 1.2 |
| 数据/训练 | Fig. 1 的多源数据配方 | 3 |
| 任务/环境、主结果 | Fig. 4–7 | 5 |
| 消融/规模 | Fig. 8–9 | 5 |

## 1. 模型原理总览：输入、输出与模块分工

| 接口 | 内容 | 未披露/注意点 |
|---|---|---|
| 输入 | 多视角 RGB 观测 $o_t$、语言指令 $l$、动作训练序列 | 论文没有把所有机器人本体的统一动作维度、单位和控制频率完整写成通用接口。 |
| 中间量 | 场景语义 $h_{und}$、未来视觉 latent/预测帧 | 未来预测是训练和推理表征的一部分，不应误读为已验证的安全规划器。 |
| 输出 | 连续 action chunk | 动作专家以 Flow Matching 解码；实际限幅、坐标系和底层控制仍随本体而变。 |

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/samples/internvla-a1-fig2-framework.png' | relative_url }}" alt="InternVLA-A1 三专家框图：理解、生成和动作通过统一掩码自注意力连接" /><figcaption><strong>论文 Fig. 2（裁切）。</strong>覆盖：模型总体、输入输出、动作表示。左侧理解图像和语言；中间预测未来视觉；右侧将共享语义和预测动态转成连续动作。</figcaption></figure>

| 模块 | 做什么 | 为什么需要它 |
|---|---|---|
| Understanding Expert | 编码图像和文本为共享上下文 | 保留 VLM 的对象、关系与指令语义。 |
| Generation Expert | 预测任务相关的未来视觉状态 | 为“此动作之后会怎样”提供动态线索。 |
| Action Expert | 从噪声动作解码连续控制 chunk | 用连续生成避免把精细控制完全离散 token 化。 |
| Unified masked self-attention | 控制三个专家可见的信息流 | 让语义、预测和动作共享上下文，同时保持各自目标。 |

**从输入到输出**：1）图像和语言进入理解专家；2）得到的语义上下文被 generation/action 分支读取；3）生成分支学习未来视觉；4）动作分支将预测动态与当前语义条件化；5）Flow Matching 从噪声恢复 action chunk，闭环中仍须重新观测。

## 2. 为什么它值得放进 WAM 线

反应式 VLA 可在“抓取什么”上很强，却未必显式学习传送带、接触或物体运动的后果。A1 的设计选择是把未来视觉作为辅助任务：它不是先生成一整段视频再做显式 MPC，而是让未来预测塑造共享表征，再由动作专家直接出控制。因此它属于 **prediction-augmented VLA**，不是完整的可验证世界模拟器。

## 3. 数据、训练与目标

论文将合成机器人轨迹、真实机器人数据和人类第一视角视频混合预训练；其中 InternData-A1 提供大规模可组合仿真，真实数据补物理真实性，人类视频补视觉先验。论文披露 A1 使用 InternData-A1 与 Agibot-World 的混合数据，覆盖 533M+ 帧。

动作分支的核心可抽象为条件 Flow Matching：

$$
\mathcal{L}_{FM}=\mathbb{E}_{a,\epsilon,\tau}\left[\left\|v_\theta(a^\tau,\tau\mid h_{und},h_{gen})-(a-\epsilon)\right\|_2^2\right].
$$

其中 $a$ 是真值连续动作，$\epsilon$ 是噪声，$a^\tau$ 是插值后的带噪动作，$h_{und}$ 与 $h_{gen}$ 分别是语义和预测动态条件，$v_\theta$ 是动作专家的速度场。直觉是：不是直接回归一个单点动作，而是学习把噪声轨迹推回多模态示范轨迹。

## 4. 与 A1.5 的关系

| A1 | A1.5 的直接改进 |
|---|---|
| 三个专家中显式学习未来视觉生成 | 改为 **foresight tokens 查询未来 latent**，由冻结的视频生成模型监督，避免在线像素生成负担。 |
| 统一 MoT 使多目标协作，但主干语义可能受干扰 | 使用原生 VLM + 轻量统一专家，继续 VQA/子任务训练以保留语义。 |
| 未来生成参与系统 | 推理时删除视频分支，仅保留动作路径，降低延迟。 |

## 5. 证据、边界与 WAM Q&A

论文在真实静态/动态任务及 RoboTwin 2.0 上比较基线，并用预训练和 generation expert 的消融检查两个设计。正确读法是：这些证据支持“未来预测辅助有收益”，但不能推出任意场景下未来帧都物理正确。

**Q：世界状态是什么？** A：主要是图像/视觉 latent 与共享多模态表示；没有显式可编辑的物体状态图。  
**Q：动作如何影响预测？** A：动作专家与生成专家的具体条件路径由 MoT/共享注意力耦合；应从论文实现继续核对时序和 action horizon。  
**Q：预测怎样帮助决策？** A：作为 action expert 的条件，而不是论文中完整公开的候选轨迹 MPC。  
**Q：主要风险？** A：视觉上合理不等于接触动力学正确；预测误差、动作时戳和本体标定误差都会在真机放大。

## 6. 算力与硬件需求（学习与部署规划）

> 范围：以下用于资源边界判断，不代表已在本机或 SO-ARM101 上验证。

| 场景 | 配置假设 | GPU/显存 | 证据等级 | 风险 |
|---|---|---|---|---|
| 全量预训练 | 2B/3B、533M+ 帧、多目标训练 | 多 GPU；官方未公开完整 GPU-hours | 论文披露 + 未知 | 单张 4090 不适用，数据吞吐与激活显存同样是瓶颈。 |
| 全参微调 | 3B、Adam、视频/动作序列 | 24GB 仅是权重下界之外仍明显不足的高风险配置 | 显存下界估算 | optimizer、梯度和激活会远超权重；不要把 24GB 当可用结论。 |
| 参数高效微调 | 冻结主干/短序列、小 batch、LoRA | 4090 24GB **可能用于离线小规模探索** | 显存下界估算 | 官方尚未给出 SO-ARM101 配方；需先验证数据与动作接口。 |
| 离线推理 | action-only 或短样本检查 | 24GB | 未知 | 吞吐不等于安全闭环频率。 |

显存由权重精度、梯度、优化器状态、激活和视频/图像/动作上下文共同决定；2B BF16 权重本身约 4GB，3B 约 6GB，但训练并不只加载权重。系统侧至少规划 64GB RAM、快速 NVMe 与独立数据备份；多机预训练还需要高速互联。对个人学习而言，应先做离线数据校验与仿真评估，不把模型输出直接连到机械臂。

## 7. 来源

- [论文](https://arxiv.org/abs/2601.02456)；本地归档：`WAM/论文/InternVLA-A1-2601.02456.pdf`。
- [官方代码与模型入口](https://github.com/InternRobotics/InternVLA-A-series)。
- [InternData-A1](https://huggingface.co/datasets/InternRobotics/InternData-A1)。
