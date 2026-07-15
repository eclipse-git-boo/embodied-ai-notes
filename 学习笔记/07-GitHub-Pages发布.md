---
layout: page
title: 发布学习笔记到 GitHub Pages
---

仓库已包含 `_config.yml`、根目录 `index.md` 和 `.github/workflows/pages.yml`。工作流会在 `main` 分支推送后构建并部署整个仓库，因此 Markdown 笔记与 PDF 都可在网页浏览。

## 一次性发布步骤

1. 在 GitHub 新建一个空仓库，例如 `embodied-ai-notes`。
2. 在本地将当前目录初始化为 Git 仓库、提交并推送到远程的 `main` 分支。
3. 打开仓库 **Settings → Pages**，将 **Source** 设为 **GitHub Actions**。
4. 查看 **Actions → Deploy GitHub Pages**；首轮成功后，站点地址会显示在该 workflow 的 deployment 输出中。

以后修改或新增笔记，只需推送到 `main`，站点会自动更新。

## 发布前检查

- PDF 已在 `VLA/论文/` 和 `WAM/论文/`；当前总量适合 GitHub 仓库。若日后加入模型权重或大数据集，必须使用 Hugging Face、对象存储或 Git LFS，**不要**直接提交权重到 Pages 仓库。
- 仓库根目录的 `.gitignore` 已默认排除权重、数据集、视频、日志和 `.env` 文件；首次提交前仍应执行 `git status` 人工检查。
- Pages 是公开静态站点。不要上传串口路径、家庭地址、摄像头原始隐私视频、令牌、API key 或真实设备密码。
- 本工作流需要默认分支名为 `main`；若使用其它分支，修改 `.github/workflows/pages.yml` 的触发分支。

## 当前限制

当前目录没有 Git 远程仓库，也没有 GitHub 账号授权信息，因此这里只能准备发布配置，不能代为创建远程仓库或启用 Pages。
