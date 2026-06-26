---
title: 不是字段太多，而是抽象错了
date: 2026-06-26
slug: llm-trace-field-design-critique
---

## 背景

我们现在讨论的是客户端发给后端的一套 LLM 请求标识字段。

产品场景并不复杂：用户新建一个会话，在这个会话里连续发送多轮消息。每一轮消息背后，Agent 可能会触发多次内部调用，例如主模型调用、RAG、工具调用、aApp 调用、结果总结、记忆压缩等。

这个场景需要表达的请求关系只有三层：

```text
Session：一个用户会话
  Message：用户发出的一轮消息
    Call：这一轮消息内部触发的一次具体调用
```

这已经足够清楚：

```text
哪些请求属于同一个会话？
哪些调用属于同一轮消息？
某一次具体调用是什么？
如果需要，某个调用是谁触发的？
```

因此，理想模型就应该是：

```text
Session -> Message -> Call
```

最多在下一层扩展一个可选的：

```text
parent_call_id
```

用来表达调用树。

但当前系统没有守住这个简单模型，而是引入了 `session_key`、`lifecycle_id`、`trace_id`、`span_id`、`parent_span_id` 等一堆字段。问题不只是字段多，而是抽象层级错了。

> [!WARNING]
> 这不是命名风格问题，而是请求追踪模型没有被认真设计之后造成的复杂性扩散。

## 理想模型

我们需要的标准非常简单。

### Session

`Session` 表示一个产品会话。

它回答的问题是：

```text
这些请求是不是属于同一个会话？
```

理想字段：

```text
session_key
```

这个字段应该只表达会话身份，不应该混入入口、aApp、feature 或 plugin 信息。

这里需要说明一个产品事实：用户确实可以从某个 aApp 入口进入一个会话。这个会话在产品体验上可以叫 aApp 会话，因为它一开始就带着一组预定义提示词、预定义功能或插件配置。

但底层模型仍然是 Chat 会话，只是这个会话被注入了某个 aApp、某组 plugin 或某组能力约束。

同样，一个普通 Chat 会话里也可以临时调用一个或多个 aApp。所以普通会话和所谓 aApp 会话不是两个互斥的底层会话类型。它们的差异主要是初始上下文和能力配置。

aApp 可以影响会话初始状态，但不应该改变底层会话身份。

### Message

`Message` 表示用户发出的一轮消息。

它回答的问题是：

```text
这些内部调用是不是由同一轮用户消息触发的？
```

理想字段：

```text
message_id
```

不需要再引入 `turn` 这个概念。`Message` 已经足够表达这一层。多引入一个 `turn`，只会让原本清楚的三层模型继续变复杂。

### Call

`Call` 表示一轮消息内部的一次具体调用。

例如：

```text
主 Agent LLM 调用
RAG answer 调用
工具参数生成调用
aApp action feedback 调用
memory flush 调用
```

理想字段：

```text
call_id
```

如果确实需要表达调用树，再增加：

```text
parent_call_id
```

注意，`parent_call_id` 是下一步扩展，不是第一版必须有的东西。只有当我们真的需要回答“这个调用是谁触发的”时，它才有价值。

## 当前模型

当前系统实际暴露的是：

```text
session_key
lifecycle_id
trace_id
span_id
parent_span_id
```

这套字段看起来很专业，但实际非常混乱。它把一个本来只有三层的请求关系，包装成了 tracing 术语集合。

当前看起来像这样：

```text
Session -> Lifecycle -> Trace -> Span
```

但我们真正需要的是：

```text
Session -> Message -> Call
```

这就是根本问题。

## 问题一：`session_key` 混入了 aApp 维度

当前某些路径会把 `session_key` 拼成：

```text
chat:<sessionId>
aapp:<aappId>:session:<sessionId>
```

这等于把“会话身份”和“入口 / aApp 维度”混在了一起。

这是建模错误。

aApp 可以是产品入口，也可以是一组预置提示词、预置能力或插件配置。但它不应该成为底层 session identity 的一级命名空间。

从 aApp 进入的会话，本质上是“带着某个 aApp 上下文启动的 Chat 会话”。从普通 Chat 进入的会话，也可以在过程中调用一个或多个 aApp。

因此，aApp 和 Chat 不是两个互斥的底层会话类型。

aApp 信息应该放在：

```text
feature
surface
entities.aappId
```

而不是放进 `session_key` 的一级结构里。

如果同一个底层会话因为入口不同或调用了不同 aApp，就被拆成不同的 session key，那么这个字段就没有忠实表达会话身份。

## 问题二：`lifecycle_id` 命名严重误导

当前主 Chat 链路里，`lifecycle_id` 基本对应一轮用户消息的 `messageRequestID`。

那它就应该叫：

```text
message_id
```

而不是：

```text
lifecycle_id
```

`lifecycle` 是一个过大的词。它可能让人以为这是 App 生命周期、Agent 生命周期、会话生命周期、任务生命周期，或者某个更大的运行周期。

但它实际表达的只是“一轮用户消息”。

这不是小的命名瑕疵，而是误导接入方理解系统层级。

## 问题三：`trace_id` 在当前场景没有独立意义

`trace_id` 来自通用 tracing 模型，理论上表示一整条链路。

但在我们的场景里，一轮用户消息天然就是一条完整的内部调用链。

如果要标识这条链，直接使用：

```text
message_id
```

就够了。

`trace_id` 试图再引入一个“链路层”，但这个层没有表达任何新的请求关系，也没有提供比 `message_id` 更清楚的分组边界。

所以它不是“可以有也可以没有”，而是一个重复层级。

它让同一件事出现两个名字：

```text
message_id：这一轮用户消息
trace_id：这一轮用户消息触发的链路
```

在我们的请求追踪标准里，这两个边界是同一个边界。保留两个名字，只会制造混乱。

## 问题四：`span_id` 其实就是 `call_id`

`span_id` 也是 tracing 术语。

但在我们的场景里，它实际表达的是：

```text
call_id
```

也就是一轮消息内部的一次具体调用。

如果它就是 call，就应该叫 call。让后端先理解 span，再把 span 映射成 call，是没有必要的心智负担。

字段名应该服务于当前系统，而不是服务于某套通用术语的完整性。

## 问题五：`parent_span_id` 看起来像调用树，实际不是

理论上，`parent_span_id` 应该表达调用树。

比如：

```text
main agent call A
  rag call B
```

如果 `rag call B` 是由 `main agent call A` 触发的，那么 `B.parent_call_id = A`。这才是有意义的 parent。

但当前实现并不是这样。

现在代码里大致是：先创建一个 `rootTrace`，然后很多真实 LLM call 都通过 `createChildLlmTrace(rootTrace, ...)` 生成。这个函数会机械地设置：

```text
parent_span_id = rootTrace.spanId
```

结果是很多调用都挂到同一个 root span 下面。

但这个 root span 通常不是一次真实发给后端的 LLM call。它只是客户端内存里的虚拟根节点。

所以后端看到的更像是：

```text
Virtual root R
  main agent call A
  rag call B
  tool call C
  memory flush call D
```

而不是：

```text
main agent call A
  rag call B
```

这并没有说明 `rag call B` 是由 `main agent call A` 触发的，只说明它们都挂在同一个虚拟 root 下。

而“它们属于同一轮用户消息”这件事，本来 `message_id` 就已经能表达。

所以当前的 `parent_span_id` 看起来像是在表达调用树，实际上没有表达真实调用树。

它不是 `parent_call_id`，只是一个名义上的 parent。

## 问题六：这是把通用 tracing 方案生搬硬套

通用 tracing 模型通常是：

```text
Trace
  Span
    Child Span
```

这套模型适合跨服务、跨进程、跨组件排查耗时和错误。

但我们这里不是在设计通用分布式 tracing 平台，而是在为当前 LLM 请求定义请求追踪标准。

当前场景需要的是：

```text
Session
  Message
    Call
```

把 `Trace -> Span -> Child Span` 原封不动搬进来，没有先判断当前场景到底需要什么层级，这是典型的复杂性搬运。

通用方案不是不能参考，但必须裁剪。

没有裁剪的通用方案，不是架构能力，是抽象懒惰。

## 问题七：字段名完全不自解释

好的字段名应该让接入方不用翻代码、不用问人，也能大致知道它该怎么用。

但现在这些字段做不到：

```text
lifecycle_id
```

看不出它到底是哪种 lifecycle。

```text
trace_id
```

看不出它对应 session、message，还是某次调用链。

```text
span_id
```

对当前后端来说不自然。真正需要的是 call。

```text
parent_span_id
```

暗示有调用树，但当前实现并没有真实调用树。

这些名字不是中性的。它们会误导接入方以为系统里存在更多层级。

如果一个字段需要解释半天才知道能不能用，它就不是一个好字段。

## 应该收敛成什么

理想字段体系应该是：

```text
session_key
message_id
call_id
```

如果下一步确实需要调用树，再增加：

```text
parent_call_id
```

也就是说：

```text
session_key：标识同一个会话
message_id：标识同一轮用户消息
call_id：标识这一轮消息内部的一次具体调用
parent_call_id：可选，标识真实调用树
```

不需要 `turn_id`。

不需要 `trace_id`。

不应该把 `span_id` 暴露成主要概念。

当前的 `parent_span_id` 也不能当作可靠的 `parent_call_id`。

## 给后端的解释

如果后端现在必须接入现有字段，可以先这样理解：

```text
x-remio-session-key / remio_session_key
= 当前用于判断同一个会话。

x-remio-lifecycle-id / remio_lifecycle_id
= 当前实际表示一轮用户消息，应该理解为 message_id。

x-remio-span-id / remio_span_id
= 当前实际表示一次具体调用，应该理解为 call_id。

x-remio-parent-span-id / remio_parent_span_id
= 名义上表示父调用，但当前多数只是挂到虚拟 root span 上，不能可靠表达真实调用树。

x-remio-trace-id / remio_trace_id
= 当前场景下没有独立必要性。它和 message_id 表达的是同一轮用户消息边界。
```

这只是兼容解释，不是理想设计。

## 结论

这套设计最大的问题，不是字段数量，而是没有守住最基本的请求关系模型。

我们需要的是：

```text
Session -> Message -> Call
```

当前却暴露成：

```text
Session -> Lifecycle -> Trace -> Span
```

这让一个本来非常清楚的问题，变成了一套需要反复解释的术语谜题。

更严重的是，这不是自然演化出来的复杂性，而是把通用 tracing 概念不加裁剪地搬进当前场景。

好的工程设计不是把所有看起来专业的字段都加上去。好的工程设计是先判断系统到底需要表达什么，然后用最少、最稳定、最贴近当前场景的概念表达它。

在这里，答案很简单：

```text
Session
Message
Call
```

其他没有真实必要性的层级，都应该收掉。
