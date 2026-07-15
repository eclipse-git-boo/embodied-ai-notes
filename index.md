---
layout: home
title: 具身智能学习笔记
---

<section class="hero">
  <div class="eyebrow">Embodied AI · Study System</div>
  <h1>从论文理解到<br>π₀.₅ 微调复现</h1>
  <p>围绕 VLA、WAM 与真实机器人开发闭环构建的中文学习资料库。当前以读懂论文、设计可复现实验为主，不在本机执行训练或真机控制。</p>
</section>

<section class="home-section">
  <h2>推荐从这里开始</h2>
  <div class="card-grid">
    <a class="nav-card" href="学习笔记/06-pi05微调复现路线.html">
      <span class="card-tag">主线 · P0</span>
      <strong>π₀.₅ 微调复现路线</strong>
      <span>从官方 LIBERO 基线，到自有数据映射、评测与部署安全。</span>
    </a>
    <a class="nav-card" href="学习笔记/04-六周学习计划.html">
      <span class="card-tag">路线 · 6 周</span>
      <strong>六周学习计划</strong>
      <span>按 ACT、RT、OpenVLA、π₀、WAM 的依赖关系推进。</span>
    </a>
    <a class="nav-card" href="学习笔记/08-论文索引与开源状态.html">
      <span class="card-tag">选题 · 论文地图</span>
      <strong>论文索引与开源状态</strong>
      <span>区分“有论文”“有代码”和“能在自己设备复现”。</span>
    </a>
    <a class="nav-card" href="学习笔记/09-pi05复现前自测.html">
      <span class="card-tag">检查 · 12 题</span>
      <strong>π₀.₅ 复现前自测</strong>
      <span>检验动作接口、数据时间对齐、评测与安全理解。</span>
    </a>
  </div>
</section>

<section class="home-section">
  <h2>按主题浏览</h2>
  <div class="card-grid">
    <a class="nav-card" href="学习笔记/论文笔记/VLA/index.html">
      <span class="card-tag">VLA / VA · 9 篇</span>
      <strong>VLA 论文笔记</strong>
      <span>OpenVLA、π₀、π₀.₅、FAST 与 LingBot VLA/VA 系列的精读报告。</span>
    </a>
    <a class="nav-card" href="学习笔记/论文笔记/WAM/index.html">
      <span class="card-tag">WAM · 4 篇</span>
      <strong>WAM 论文笔记</strong>
      <span>DreamerV3、UniPi、IRASim 与 WAM Survey。</span>
    </a>
    <a class="nav-card" href="学习笔记/05-从论文到工程的知识地图.html">
      <span class="card-tag">工程 · 数据闭环</span>
      <strong>从论文到工程</strong>
      <span>理解硬件、数据契约、训练、评测之间的接口。</span>
    </a>
    <a class="nav-card" href="学习笔记/10-openpi论文到代码对照.html">
      <span class="card-tag">代码 · openpi</span>
      <strong>论文到官方代码对照</strong>
      <span>把 π₀.₅论文概念对应到 Inputs、Outputs、Config 与 server。</span>
    </a>
  </div>
</section>

<section class="home-section focus-card">
  <strong>当前聚焦：π₀.₅，而非同时复现所有模型。</strong>
  先阅读 π₀、FAST、π₀.₅与 Knowledge Insulating VLA；完成官方离线基线后，再考虑 SO-ARM101 的数据与动作适配。WAM 是后续用于策略评估或候选动作重排的扩展方向。
</section>
