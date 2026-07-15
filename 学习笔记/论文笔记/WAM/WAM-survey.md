---
layout: page
title: "World Action Models Survey：WAM 的分类与阅读地图"
description: "从 token 表示、动力学、动作生成到规划评测的 WAM 综述。"
---

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/wam-survey-02.png' | relative_url }}" alt="WAM Survey 图示"><figcaption>论文图示摘录（PDF 第 2 页）。来源：本库收录的 WAM Survey PDF。</figcaption></figure>

## 1. 快读卡片

| 项目 | 内容 |
|---|---|
| 论文 | *World Action Models Survey*（2026） |
| 作用 | 不是单一模型配方，而是把“预测世界 + 生成/选择动作”的工作放到统一坐标系中。 |
| 阅读顺序 | 先读 OpenVLA/π0 建立反应式 VLA 参照，再读 DreamerV3 → UniPi → IRASim → LingBot-VA。 |

## 2. 为什么需要 WAM

反应式策略学习 \(p(a_t\mid o_{\le t},x)\)，而 WAM 进一步显式描述动作后果，例如 \(p(o_{t+1:t+H},a_{t:t+H}\mid o_{\le t},x)\)。这种建模可支持规划、反事实比较、失败预警和从无动作视频中学习；代价是模型偏差、计算开销以及“生成视觉 ≠ 物理正确”的风险。

## 3. 综述框架

可用四个问题定位一篇论文：

1. **表示**：像素、VAE latent、语义视觉 token，还是共享 video-action token？
2. **动力学**：自回归、RSSM、diffusion / flow、因果 attention？
3. **动作角色**：动作条件、逆动力学预测、联合生成，还是用模型做候选规划？
4. **闭环机制**：开环 rollout、多步 MPC、异步执行、re-ground，还是安全 shield？

<div class="method-flow"><span>表示学习</span><b>→</b><span>动作条件动力学</span><b>→</b><span>预测 / 反事实</span><b>→</b><span>规划或策略</span><b>→</b><span>闭环安全执行</span></div>

## 4. 阅读结论与适用边界

WAM 的真正价值不是“多生成几帧视频”，而是把状态转移变成可学习、可检查的对象。DreamerV3 偏 latent control；UniPi/IRASim 偏视频预测辅助规划；LingBot-VA 则向联合 video-action 基础模型推进。对真实机器人，模型应始终被最新传感重置/校正，并置于传统低层控制与安全约束之上。

## 5. 可复现性审计清单

| 维度 | 必答问题 |
|---|---|
| 训练数据 | 是否有动作、频率是否一致、是否覆盖接触/失败？ |
| 表示质量 | 预测是否对动作敏感，而不只是画面平滑？ |
| 控制收益 | 比反应式 VLA 提高了什么：成功率、样本效率、恢复、还是延迟？ |
| 闭环 | 多久 re-ground 一次？模型超时/不确定时怎么回退？ |
| 安全 | 速度、力矩、工作空间、碰撞和急停是否独立于模型？ |

## 6. 算力、硬件与风险（学习规划，不在本机部署）

| 层级 | 建议配置 | 风险 |
|---|---|---|
| 综述复读 / toy benchmark | 单卡 12–24 GB | 先建立对比实验，不接真机。 |
| 图像 world model 后训练 | 24–48 GB + NVMe | 长序列、视频解码和 replay 很快成为瓶颈。 |
| 大规模 VA 预训练 | 多节点高显存 GPU、海量视频动作数据 | 预算/数据/评测都远超个人范围。 |
| 真实控制 | world model 只做建议或评分；独立安全层必须接管 | 世界模型偏差、长 rollout 和生成幻觉会制造高风险动作。 |

对当前计划：WAM 作为理解未来方向的支线；π0.5 的单本体后训练和闭环数据质量仍应优先。
