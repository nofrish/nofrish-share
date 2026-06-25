---
title: IWannaShare 的 Codex 使用方式
slug: iwannashare-codex-workflow
date: 2026-06-25
---

如果你是在 Codex 里使用 IWannaShare，最常见的方式不是手工整理文件，而是让 Codex 直接把一段文字、总结、笔记或对话结果变成可分享成品。

它的主要价值在于：
- 你给 Codex 一段 Markdown 或结构化内容
- Codex 直接调用 iws 生成长图或网页
- 你拿到一张可发出去的图，或者一个可转发的链接

## 在 Codex 里怎么安装

先把 skill 装进 Codex：

https://github.com/nofrish/iwannashare/tree/main/skills/iwannashare

如果只装 skill，iws 命令还需要单独可用。最直接的方式是：

    npm install -g github:nofrish/iwannashare
    iws doctor

iws doctor 会检查本地 Chrome 是否可用。只要这一步通过，后面的渲染和发布就能跑。

## 在 Codex 里怎么调用

当你要分享一段内容时，最适合的方式是让 Codex 先把正文整理成 Markdown，再直接喂给 iws。

### 生成长图

    iws --copy <<'MD'
    ---
    title: 你的标题
    date: 2026-06-25
    ---

    正文内容。
    MD

这个命令适合：
- AI 总结
- 会议纪要
- 方案草稿
- 可直接发朋友圈或群聊的长图内容

### 发布成公开链接

如果你要的是一个 URL，而不是 PNG，就让 Codex 走发布命令：

    iws publish --copy-url <<'MD'
    ---
    title: 你的标题
    slug: 你的稳定链接
    ---

    正文内容。
    MD

发布完成后，Codex 会拿到最终链接，并且可以继续帮你把这个链接整理进文档、回复、工单或知识库。

## Codex 里最推荐的工作流

1. 先让 Codex 产出 Markdown 正文
2. 再让 Codex 按目标选择 PNG 或网页
3. 如果是公开传播，优先用 iws publish --copy-url
4. 如果是临时传播或聊天分享，优先用 iws --copy

## 这个项目对 AI 同事为什么友好

这个仓库的结构比较适合 AI 接手，因为入口清晰：
- src/cli.ts 管命令
- src/renderer/ 管长图渲染
- src/web/ 管网页导出和发布
- src/config.ts 管本地发布配置
- src/git.ts 管 Git 提交和推送

所以 AI 同事只要知道目标输出是什么，就能比较直接地定位该改哪一层。

> [!NOTE]
> 这个工具最适合“先让 AI 生成内容，再由 IWannaShare 负责最后一公里”的流程。

## 你在 Codex 里可以直接说的话

你可以直接让 Codex 做这些事：
- 把这段内容整理成适合 iws --copy 的 Markdown
- 把这段总结发布成公开链接，使用 iws publish
- 把输出改成更适合手机阅读的结构
- 帮我把这篇内容改成能被同事接手的安装文档

## 一句话总结

在 Codex 里，IWannaShare 更像是一个最后输出器：Codex 负责内容，iws 负责变成图或链接。
