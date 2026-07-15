---
layout: page
title: openpi：从 π₀.₅论文到官方代码
---

官方依据：[openpi README](https://github.com/Physical-Intelligence/openpi/blob/main/README.md)（访问日期：2026-07-15）。本页只解释接口，不在当前电脑执行命令。

## 论文概念与代码对象

| 论文中的概念 | 官方代码/流程中的对应物 | 未来要做的检查 |
| --- | --- | --- |
| 多视角图像、proprioception、语言 | `Inputs`：将本体观测映射为模型输入 | 图像键/顺序、RGB、尺寸、状态单位、prompt 格式 |
| 连续 action chunk | `Outputs`：模型动作与本体命令双向映射 | 维数、joint/EEF、absolute/delta、夹爪、坐标系 |
| 自有演示数据 | LeRobot dataset | episode 边界、时间戳、训练/测试划分、元数据 |
| 数据预处理与采样 | `DataConfig` | 训练样本窗口、图像变换、语言来源、数据 revision |
| π₀.₅权重与优化超参数 | `TrainConfig` + weight loader | 基础权重、冻结策略、batch、步数、seed、checkpoint 路径 |
| 动作标准化 | `compute_norm_stats.py` | 仅使用训练集；保存 stats 和数据版本 |
| 在线策略 | `serve_policy.py` + robot runtime | 观测协议、端到端延迟、动作后处理、安全层 |

## 官方路径的四个不可跳过步骤

1. **先复现 LIBERO**：官方示例以 `pi05_libero` 演示。先成功运行相同的 config、norm stats、训练与评测，才修改数据映射。
2. **再转自有数据**：官方把 LeRobot 作为自有/小规模数据的训练格式；转换脚本是模板，不是能直接套用 SO-ARM101 的保证。
3. **先计算并固化 norm stats**：输入/动作统计量属于 checkpoint 的一部分；改数据、动作空间或划分就必须重算或有充分理由复用。
4. **通过 policy server 接入运行时**：server 只负责模型推理；机械臂驱动仍须自行负责时间同步、单位转换、限位、急停与动作执行。

## 4090 48GB 的选择

官方 README 当前列出单 GPU 推理大于 8GB、LoRA 微调大于 22.5GB、完整微调约需 70GB（A100/H100 80GB 级别）。因此 4090 48GB 的合理目标是：

- 先跑 π₀.₅ LIBERO 推理/官方小规模微调；
- 优先使用官方在目标 commit 中明确支持的低显存路径；
- 不把“可装下 LoRA”误认为“任何完整微调配方都可运行”；
- PyTorch 与 JAX 功能不同：目前官方 README 指出 PyTorch 的 π₀-FAST、mixed precision、FSDP、LoRA、EMA 有支持边界，启动前重新确认目标 commit。

## SO-ARM101/NERO 适配检查单

| 在改代码前回答 | 证据 |
| --- | --- |
| 采集到的每帧图像如何映射到 `Inputs`？ | 一条 episode 的可视化与键名表 |
| 模型动作如何映射到真实控制器？ | `Outputs` round-trip 数值测试 |
| 机器人状态的单位和参考系是什么？ | 厂商/驱动文档与低速手动验证 |
| 模型请求和控制器反馈延迟多少？ | 端到端时间戳日志 |
| 遇到越界、超时、断流如何停止？ | 独立安全层与急停演练记录 |

在以上问题没有书面答案前，不连接经过微调的策略到真机。

