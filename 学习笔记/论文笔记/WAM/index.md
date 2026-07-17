---
layout: page
title: WAM 论文精读
---

# WAM：世界预测如何服务动作

这里收录两类工作：以预测世界为核心的 WAM，以及把未来状态/潜在未来用于控制的操控 VLA。后者的主接口仍是 VLA，但对理解“预测为何能改善动作”很有价值。

1. [InternVLA-A1：理解、未来视觉与动作的统一 MoT](InternVLA-A1.html)
2. [InternVLA-A1.5：latent foresight 与可部署动作专家](InternVLA-A1.5.html)
3. [LingBot-VA：因果视频—动作世界模型](../VLA/LingBot-VA.html)
4. [LingBot-VA 2.0：预测与动作的统一接口](../VLA/LingBot-VA-2.0.html)
5. [DreamerV3：在潜空间中想象并学习控制](DreamerV3.html)
6. [UniPi：以文本条件视频生成作为通用策略](UniPi.html)
7. [IRASim：动作条件机器人视频世界模型](IRASim.html)
8. [World Action Models Survey：WAM 分类与阅读地图](WAM-survey.html)

建议先读 A1 → A1.5，理解“像素未来预测”如何演化为“训练期 latent foresight”；再读 DreamerV3、UniPi、IRASim，以区分 imagined rollout、视频规划和动作条件视频预测。
