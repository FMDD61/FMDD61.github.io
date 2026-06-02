# AGENTS.md

## 项目概述

基于 Hexo 7.3.0 + Fluid 1.9.9 主题的个人技术博客，部署在 GitHub Pages (`fmdd61.github.io`)。

## 仓库架构

```
FMDD61.github.io
├── main 分支     ← GitHub Pages 服务分支（hexo g -d 自动推送，永远不手动编辑）
└── source 分支   ← 日常工作分支（Hexo 源码、文章、配置）
```

## 目录结构（source 分支）

```
D:\blog\
├── _config.yml              ← Hexo 站点配置
├── _config.fluid.yml        ← Fluid 主题覆盖配置（需手动创建）
├── source/
│   └── _posts/              ← 文章 (.md)，Hexo front matter 格式
├── scaffolds/               ← hexo new 模板
├── themes/                  ← 主题目录（npm 安装，不提交）
├── raw-logs/                ← 排障原始日志备份（仅 source 分支，不部署）
├── package.json
├── .gitignore               ← 排除 node_modules/public/.deploy_git/db.json
└── TODO.md                  ← 待办事项
```

## 日常工作流

### 写新文章

```bash
cd D:\blog
npx hexo new "文章标题"
# 编辑 source/_posts/文章标题.md
# 确保 front matter 包含 title, date, categories, tags
npx hexo g -d
git add -A && git commit -m "新文章: 文章标题" && git push
```

### 修改配置

1. 编辑 `_config.yml` 或 `_config.fluid.yml`
2. `npx hexo clean && npx hexo g -d`（配置变更建议 clean 后重新生成）
3. `git add -A && git commit -m "配置: xxx" && git push`

### 本地预览

```bash
npx hexo server
# 浏览器打开 http://localhost:4000
```

## 关键决策

| 决策 | 结论 | 原因 |
|---|---|---|
| 主题 | Fluid | 美观、中文支持好、GPL-3.0 对博客无影响 |
| 部署方式 | `hexo-deployer-git` 推 main 分支 | GitHub Pages 标准做法 |
| 认证方式 | GitHub CLI (`gh`) | 无需 SSH 配置，浏览器 OAuth 授权 |
| raw-logs | 仅 source 分支备份 | 不部署到网站，仅作排障记录 |
| 双分支 | main + source | main 只放生成文件，source 放源码 |

## 技术要点

### Fluid 主题配置

- 主题通过 npm 安装：`npm install hexo-theme-fluid --save`
- 推荐使用覆盖配置：创建 `_config.fluid.yml`（而非直接修改 `node_modules/hexo-theme-fluid/_config.yml`）
- 覆盖配置文档：https://hexo.fluid-dev.com/docs/guide/#覆盖配置

### 文章 Front Matter 格式

```yaml
---
title: 文章标题
date: 2026-06-01 22:50:00
categories:
  - 分类1
  - 分类2
tags:
  - 标签1
  - 标签2
---
```

### hexo g -d 注意事项

- 首次部署或配置变更后建议先 `hexo clean`
- 部署只影响 main 分支，不影响 source 分支
- `public/` 和 `.deploy_git/` 已在 .gitignore 中排除

## 已知问题与教训

1. **Fluid 主题安装失败**：首次 `npm install hexo-theme-fluid` 因临时目录迁移导致未实际安装，`themes/` 为空。修复：在 `D:\blog` 下重新执行 `npm install hexo-theme-fluid --save`。
2. **hexo init 拒绝非空目录**：因 `.git` 目录存在，需在临时目录初始化后移入。
3. **LF/CRLF 警告**：Windows 下 Git 自动转换换行符，不影响功能，可忽略。
4. **No layout 警告**：主题未安装时所有页面生成为空 HTML，安装主题后恢复正常。

## 环境要求

- Node.js >= 14（当前 v24.14.0）
- npm >= 7（当前 11.9.0）
- Git
- GitHub CLI (`gh`)，已通过 `gh auth login` 授权
