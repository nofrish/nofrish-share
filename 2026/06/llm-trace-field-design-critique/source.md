---
title: 不是字段太多，而是抽象错了
date: 2026-06-26
slug: llm-trace-field-design-critique
---

## 背景

我们现在讨论的是一套 LLM 请求标识字段。

产品场景并不复杂：用户新建一个会话，在会话里连续发多轮消息。每一轮消息可能触发 Agent 内部多次调用，例如主模型调用、RAG、工具调用、aApp 调用、结果总结、记忆压缩等。

这件事从业务视角看，是一个非常清楚的三层结构：

```text
Session：一个用户会话
  Message / Turn：用户发出的一轮消息
    Call：这一轮消息内部触发的一次具体调用
```

这套结构已经足够表达核心问题：

```text
哪些请求属于同一个会话？
哪些调用属于同一轮用户消息？
某一次具体调用是什么？
如果需要，某个调用是谁触发的？
```

但当前系统引入了 `session_key`、`lifecycle_id`、`trace_id`、`span_id`、`parent_span_id` 等一组字段。问题不只是“字段多”，而是这些字段背后的抽象层级没有被认真建模。

> [!WARNING]
> 这不是命名风格问题，而是业务模型和观测模型混在一起之后造成的抽象污染。

## 理想情况下应该怎么表达

一个合理的设计，应该先从业务对象出发。

### 1. Session

`Session` 表示一个产品会话。

它的职责只有一个：标记多轮用户消息属于同一个会话。

理想字段：

```text
session_id
session_key
```

这个字段不应该携带功能入口信息，也不应该因为用户从 Chat 进入、从 aApp 进入、或在 Chat 中调起 aApp 而改变。

会话就是会话。

如果同一个 Chat 会话里调起了多个 aApp，这些 aApp 是能力、工具或上下文，不是新的会话边界。

### 2. Message / Turn

`Message` 或 `Turn` 表示用户发出的一轮消息。

一轮用户消息可能触发很多内部调用，但这些调用都属于同一个用户意图。

理想字段：

```text
message_id
message_request_id
turn_id
```

这个字段负责回答：

```text
这些内部调用是不是同一轮用户消息触发的？
```

### 3. Call

`Call` 表示一轮消息内部的一次具体调用。

比如：

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
parent_call_id（可选）
```

`parent_call_id` 只有在需要还原调用树时才有价值。如果只是做归属统计，`session_id + message_id + call_id` 已经足够。

## 当前系统实际怎么表达

当前系统使用的是：

```text
session_key
lifecycle_id
trace_id
span_id
parent_span_id
```

表面上看，每个字段都好像有来源、有概念、有工程味道。但放到我们的业务场景里看，问题非常明显。

### `session_key`

它应该表示会话。

但当前某些路径会把它拼成：

```text
chat:<sessionId>
aapp:<aappId>:session:<sessionId>
```

这等于把“会话身份”和“功能入口 / aApp 身份”混在了一起。

这是一个典型的建模错误。

因为 aApp 不是会话边界。aApp 应该出现在：

```text
feature
surface
entities.aappId
```

而不是被塞进 `session_key`。

如果同一个会话里调用多个 aApp，却因为 `session_key` 不同而被拆成多个“会话”，那么这个字段就没有忠实表达业务现实。

### `lifecycle_id`

这个字段最让人困惑。

从当前主 Chat 链路看，它基本对应一轮用户消息的 `messageRequestID`。

那它就应该叫：

```text
message_request_id
turn_id
```

而不是 `lifecycle_id`。

`lifecycle` 是一个很大的词。它暗示某种生命周期边界，但这里实际表达的只是“一轮用户消息”。名字过大，语义过虚，后续接入方就会不知道它到底应该怎么用。

### `trace_id`

`trace_id` 来自分布式 tracing 体系，理论上表示一整条链路。

在我们的场景里，它大致对应：

```text
一轮用户消息触发的内部调用链路
```

这就和 `lifecycle_id` 的含义高度重叠。

如果 `lifecycle_id` 已经代表一轮用户消息，那么 `trace_id` 作为业务字段就没有必要再出现。它可以作为观测字段存在，但不应该进入业务判断。

### `span_id`

`span_id` 也是 tracing 术语。

在我们的业务里，它其实就是：

```text
call_id
```

它表示一轮消息内部的某一次具体调用。

这个字段本身有价值，但问题在于命名把简单概念复杂化了。

业务后端需要理解的是“这是哪一次调用”，不是先学习 OpenTelemetry 式的 span 概念。

### `parent_span_id`

这个字段只有在需要调用树时才有价值。

例如：

```text
主 Agent call
  RAG call
  tool args call
    aApp feedback call
```

如果我们要知道某个失败调用是被谁触发的，或者要分析哪条调用链耗时最长，`parent_span_id` 有用。

但如果后端只需要做会话归属、消息归属和调用记录，它不是必需字段。

把可选的调用树能力和核心业务标识混在一起，会让接入方误以为每个字段都是必需的业务字段。

## 关键问题不是“字段多”，而是“层级乱”

当前设计看起来像这样：

```text
Session -> Lifecycle -> Trace -> Span
```

但业务真实结构是：

```text
Session -> Message / Turn -> Call
```

这就是问题的核心。

系统里不是天然存在一个独立的 `Lifecycle` 层，也不是每个后端逻辑都需要理解 `Trace` 和 `Span`。

`Trace / Span` 是观测系统的语言。

`Session / Message / Call` 才是业务系统的语言。

当前设计把观测语言提前暴露给业务接口，导致业务后端不得不理解一堆并不服务于业务判断的概念。

## 这类问题为什么危险

这类字段设计看起来只是“多加了几个 header”，实际上会制造长期成本。

### 1. 接入方不知道该信哪个字段

当系统同时给出：

```text
session_key
lifecycle_id
trace_id
span_id
```

后端接入方会自然追问：

```text
哪个代表会话？
哪个代表一轮消息？
trace 和 lifecycle 是什么关系？
span 是不是一次 message？
parent span 我是不是必须处理？
```

这些问题本来不应该存在。

好的接口设计应该让接入方一眼知道字段职责。

### 2. 业务模型被 tracing 模型反向绑架

Tracing 是为排障服务的，不是为定义业务对象服务的。

如果业务后端需要先理解 trace/span 才能判断一组请求的关系，说明接口抽象已经偏离业务了。

### 3. 字段一旦进入协议，就很难删除

header 和 body 字段一旦发给后端，就会被日志、统计、路由、监控、排障系统依赖。

早期随手加的字段，后面会变成协议债务。

所以字段不是越多越安全。没有清晰语义的字段越多，系统越脆。

### 4. AI 生成代码不能替代工程判断

这类设计很像“看到 tracing 概念就全塞进去”的结果：trace、span、parent span、lifecycle、session，看起来都专业，但没有先问一个最基本的问题：

```text
我们的业务场景到底需要表达几层关系？
```

AI 很擅长生成一套看起来完整的工程术语，但它不会自动保证这些术语适合当前业务。

工程师的职责不是把 AI 生成的概念照单全收，而是判断：

```text
这个抽象是否必要？
这个字段是否有唯一职责？
它是否符合我们的业务边界？
未来接入方能不能理解？
```

如果这些问题没有回答清楚，就把字段发进协议里，本质上是在把未经审查的复杂性扩散给后端和未来维护者。

## 更合理的字段体系

我们应该把字段重新映射到业务三层结构。

### 核心业务字段

```text
session_key
```

表示同一个产品会话。

```text
message_request_id / turn_id
```

表示一轮用户消息。

```text
call_id
```

表示这一轮消息内部的一次具体调用。

```text
parent_call_id
```

可选。只有需要调用树、耗时链路、错误传播路径时使用。

### 观测字段

```text
trace_id
span_id
parent_span_id
```

可以保留给日志和 tracing 系统，但不要让业务后端必须依赖它们理解会话和消息。

如果保留，也应该明确文档化：

```text
trace_id 只用于 observability
span_id 可映射为 call_id
parent_span_id 可映射为 parent_call_id
```

## 当前字段应该怎么解释给后端

如果后端现在要接入，建议这样解释：

```text
x-remio-session-key / remio_session_key
= 判断同一个会话的唯一优先字段。

x-remio-lifecycle-id / remio_lifecycle_id
= 当前表示一轮用户消息，等价理解为 message_request_id / turn_id。

x-remio-span-id / remio_span_id
= 当前表示一次具体内部调用，等价理解为 call_id。

x-remio-parent-span-id / remio_parent_span_id
= 当前表示父调用，只有需要调用树时才处理。

x-remio-trace-id / remio_trace_id
= 观测字段，用于日志聚合和排障，不建议作为业务归属判断依据。
```

## 结论

这套设计最大的问题，不是字段数量，而是没有守住业务抽象。

业务上只有三层：

```text
Session -> Message / Turn -> Call
```

当前却暴露成了：

```text
Session -> Lifecycle -> Trace -> Span
```

这让一个本来清晰的会话模型变成了一套需要解释半天的 tracing 术语集合。

好的工程设计不是把所有看起来专业的字段都加上去。好的工程设计是先判断系统真实需要表达什么，再选择最少、最稳定、最贴近业务的概念。

在这个场景里，最合理的表达应该是：

```text
session_key：会话
message_request_id / turn_id：一轮用户消息
call_id：一次内部调用
parent_call_id：可选调用树关系
```

其他字段可以作为观测辅助，但不应该成为业务协议的核心。

如果一个字段需要长篇解释接入方才知道怎么用，它就已经不是一个好字段。
