---
layout: page
title: World Action Models Survey：WAM分类与阅读地图
---

# World Action Models Survey：WAM 分类与阅读地图

## 快速卡片

| 项目 | 内容 |
|---|---|
| 论文 | [World Action Models: A Survey](https://arxiv.org/abs/2605.12090) |
| 官方来源 | [arXiv 论文页](https://arxiv.org/abs/2605.12090) |
| 最佳技术解读 | 该工作本身就是面向分类和路线的综述；建议以作者的分类图为主，而非把新闻稿当技术解读。 |
| 核心定义 | WAM 是同时建模世界变化与动作生成/选择的具身模型；关键差别在 world/action 是联合生成还是级联，以及表示是显式像素/视频还是隐式 latent。 |

## 1. 模型原理总览：输入、输出与模块分工

| 类型 | 输入 | 输出 | 决策角色 |
|---|---|---|---|
| Joint WAM | 观测、历史、任务条件 | 世界 token 与动作 token 联合序列 | 直接联合预测或共同表征 |
| Cascaded WAM | 观测、候选动作/任务 | 未来世界预测，再由 policy/planner 选动作 | 先预测后评估/规划 |
| 显式 WAM | 图像/视频与动作 | 可视未来帧或轨迹 | 易解释但昂贵、长时易漂移 |
| 隐式 WAM | latent 状态与动作 | latent rollout、价值/动作 | 高效但可解释性和校准更难 |

| 阅读模块 | 要问的问题 |
|---|---|
| world representation | 预测的究竟是像素、视频 token、3D/状态还是 latent？ |
| action interface | 动作是连续 chunk、离散 token、waypoint，还是高层子任务？ |
| coupling | 世界与动作由一个序列联合预测，还是世界模型服务外部 policy？ |
| decision use | 预测是否实际用于候选重排、规划、价值学习或安全过滤？ |

### 图谱：六类证据图

| 图 | 覆盖类别 | 用途 |
|---|---|---|
| Fig. 1 | 模型总览；数据/训练；消融/规模/效率 | Joint/Cascaded、显式/隐式与时间演进 |
| Fig. 2 | 模型总览；数据/训练 | 全文阅读路线和方法树 |
| Fig. 3 | 输入输出/动作表示 | WAM 与传统 VLA/world model 的概念边界 |
| Fig. 4 | 任务/环境；主结果 | WAM 如何服务模仿、规划与 VLA 评测 |

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/samples/wam-survey-fig1-taxonomy.png' | relative_url }}" alt="WAM时间演进和分类图" /><figcaption><strong>论文 Fig. 1（裁切）。</strong>左侧是 Joint WAM 的统一流/多流和自回归/扩散分支，右侧是 Cascaded WAM 的显式/隐式路线。它是本领域的第一张阅读地图。</figcaption></figure>

### 用这张综述读一篇 WAM 的 5 步

1. 写清楚观测、动作、任务条件和预测 horizon。
2. 判断世界表示是可视（视频/图像）还是隐式（latent/state）。
3. 判断动作和世界 token 是否联合生成，或是两阶段级联。
4. 找到预测进入决策的证据：候选排序、规划成功、policy 改善还是仅有生成质量。
5. 分开记录世界预测指标、决策指标、真实/仿真协议和算力，防止“画面好看=策略好”的误读。

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/samples/wam-survey-fig2-roadmap.png' | relative_url }}" alt="WAM综述路线图" /><figcaption><strong>论文 Fig. 2（裁切）。</strong>综述按表示、耦合方式、训练数据和应用组织文献。读新论文时先在这张路线图定位，再比较同一支路的协议。</figcaption></figure>

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/samples/wam-survey-fig3-definition.png' | relative_url }}" alt="WAM概念定义与输入输出比较" /><figcaption><strong>论文 Fig. 3（裁切）。</strong>WAM 的关键不只在“有个世界模型”，而在世界预测与动作生成/选择的结构耦合；该图用 I/O 边界帮助避免术语泛化。</figcaption></figure>

<figure class="paper-figure"><img src="{{ '/assets/paper-figures/samples/wam-survey-fig4-vla.png' | relative_url }}" alt="WAM服务VLA学习和评测的示意" /><figcaption><strong>论文 Fig. 4（裁切）。</strong>WAM 可服务 imitation、候选动作评估、latent planning 与 VLA 评测。它是任务/应用证据，不是证明任一具体模型已经安全可部署的结果表。</figcaption></figure>

## 2. 统一抽象与局限

可以用联合分布描述 WAM 的目标：

$$
p_\theta(o_{t+1:t+H},a_{t:t+H-1}\mid o_{\le t},x).
$$

Joint WAM 直接学习这类联合/交织序列；cascaded 模型则将其分解为世界预测与动作选择。此式不是所有方法的实际 loss，而是比较输入、输出和条件依赖的共同语言。

**领域局限。** 数据中的动作—后果相关性可能被相机运动或任务偏差伪造；长 horizon 误差、生成成本和现实安全仍是瓶颈。综述不是实验论文，因此不应从它抽取某一模型的性能数字。

## 3. 算力/硬件审计与 WAM Q&A

| 场景 | 综述告诉我们的事实 | 学习规划（个人推断） |
|---|---|---|
| 全量训练 | 成本由表示决定：像素/视频扩散通常比 latent rollout 更重，联合序列也增大上下文成本 | 先选小型 latent world model 理解闭环；视频 WAM 全量训练通常需多 GPU 与高速视频存储 |
| 微调 | 需要保持动作、相机、时间戳和世界状态语义一致 | 24GB 可做小环境/短 horizon 原型；48GB 以上更适合视频/多视角与较长序列 |
| 推理/规划 | 候选数、horizon 与采样步数共同决定时延 | 先记录候选数×预测时长×模型调用次数，离线仿真验证再谈控制 |

**Q：一篇 WAM 论文最重要的三项证据？** 动作条件预测是否正确、预测是否改善决策、在模型失配/OOD 下是否报告失败与不确定性。

**Q：与 VLA 学习主线怎么衔接？** 先掌握 π0.5 类直接 policy 的数据/动作接口，再把 WAM 当成预测、重排和规划补充；不要同时把二者作为第一套真机复现对象。

## 来源账本

- **论文事实：** [arXiv](https://arxiv.org/abs/2605.12090)，Fig. 1–4。
- **辅助解读：** 综述论文自身承担分类解读；此笔记只做原创归纳，不迁移第三方长文或图片。
