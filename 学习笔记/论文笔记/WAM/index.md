---
layout: page
title: WAM 论文精读
---

# WAM 论文精读

WAM（World Action Model）关注“采取动作后世界会怎样变化”，以及该预测怎样真正帮助策略或规划。阅读每篇时都先写清输入、动作、世界表示、预测 horizon 和决策使用方式，再看指标。

1. [DreamerV3：在潜空间中想象并学习控制](DreamerV3.html)
2. [UniPi：以文本条件视频生成作为通用策略](UniPi.html)
3. [IRASim：动作条件机器人视频世界模型](IRASim.html)
4. [World Action Models Survey：WAM 分类与阅读地图](WAM-survey.html)

推荐顺序：先读 DreamerV3 理解 latent imagination；再读 UniPi、IRASim 对比视频计划与动作条件视频预测；最后用综述定位 Joint/Cascaded、显式/隐式等术语。

> 这是一条论文学习支线，与 π0.5 微调复现主线分开推进：先离线评估世界预测是否改善决策，再讨论任何硬件与部署问题。
