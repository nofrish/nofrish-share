---
title: IWannaShare 安装与上手指南
slug: iwannashare-install-and-getting-started
date: 2026-06-25
---

IWannaShare 是一个本地优先的 Markdown 分享工具，主要做两件事：把文本渲染成适合手机阅读的长图，或者导出成可公开发布的静态网页。

它适合这类场景：
- 你想把一段 Markdown、AI 回复、笔记快速分享出去
- 你想生成一张适合微信传播的长图
- 你想把同一份内容发布成一个稳定 URL
- 你希望 AI 同事也能沿着仓库代码继续扩展功能

## 它是什么

项目的核心流程很简单：

    Markdown -> HTML/CSS -> 本地 Chrome 截图 -> PNG
    Markdown -> 静态 HTML/CSS -> Git 提交/推送 -> 公网 URL

所以它不是在线 SaaS，而是一个依赖本地环境和 GitHub Pages 仓库的 CLI 工具。

## 安装

### 方式一：直接从 GitHub 安装

```bash
npm install -g github:nofrish/iwannashare
iws doctor
```

这会安装 `iws` / `iwannashare` 命令。`iws doctor` 用来检查本地浏览器是否可用。

### 方式二：从源码仓库安装

```bash
git clone https://github.com/nofrish/iwannashare.git
cd iwannashare
./scripts/install.sh
```

这个方式会同时安装 CLI 和 Codex Skill。

## 运行前提

- Node.js 20 或更高版本
- Git
- Chrome 或兼容的 Chromium 浏览器
- 如果要发布网页，还需要一个单独的 GitHub Pages 仓库

## 本地开发

先拉依赖：

```bash
npm install
```

检查环境：

```bash
npm run doctor
```

渲染示例：

```bash
npm run render:example
```

你也可以直接渲染自己的 Markdown：

```bash
npm run dev -- ./my-note.md --output ./share.png --open
```

如果是从标准输入输入内容，也可以这样：

```bash
npm run dev -- --copy <<'MD'
---
title: 我的分享
---

这里放正文。
MD
```

## 生成长图

最常用的命令是：

```bash
iws ./my-note.md --output ./share.png --copy --open
```

说明：
- `--copy` 会把 PNG 复制到 macOS 剪贴板
- `--open` 会自动打开结果图
- 不指定 `--output` 时，默认会写到 `share.png`

## 发布成博客链接

先配置本地发布信息：

```bash
iws config init \
  --repo-path ~/Documents/share.example.com \
  --base-url https://share.example.com \
  --content-dir "" \
  --site-name IWannaShare
```

然后发布：

```bash
iws publish ./my-note.md --copy-url --open
```

发布时它会：
- 解析 Markdown
- 生成静态页面
- 写入 GitHub Pages 仓库
- 提交并推送到远端
- 输出最终公开 URL

## 给 AI 同事的使用建议

如果是 AI 同事来接手，建议按这个顺序看：

1. 先读 `README.md`，理解 CLI 面向的两个输出：长图和网页
2. 再看 `src/cli.ts`，这里定义了全部命令入口
3. 看 `src/renderer/`，理解长图是怎么渲染出来的
4. 看 `src/web/`，理解网页导出和发布流程
5. 看 `src/config.ts` 和 `src/git.ts`，理解发布依赖的本地配置和 Git 操作

这个项目最重要的约束是：
- 发布不是“直接发到本仓库”
- 而是发到一个单独的 GitHub Pages 仓库
- 目标仓库必须是干净工作区，且在配置指定的分支上

## 你可以怎么继续扩展

如果你要让 AI 同事继续做更多事，最值得扩展的方向通常是：
- 新主题和排版方向
- 更多 Markdown 语法支持
- 发布流程自动化
- 增加更稳的浏览器检测和错误提示
- 让 web export 的目录结构更适合静态站点托管

## 一句话总结

IWannaShare 的定位不是“写文档的工具”，而是“把文本稳定地变成可分享成品”的本地工作流。