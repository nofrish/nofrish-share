---
title: 会话、消息与调用：重新梳理 LLM 请求标识体系
date: 2026-06-26
slug: llm-session-message-call-identifiers
---

## 背景

我们现在的产品形态，本质上是一个用户和 Agent 进行多轮对话。

用户可以新建一个会话，然后在这个会话里连续发送多轮消息。每一轮用户消息背后，Agent 内部可能会触发很多动作：主模型调用、RAG 调用、工具调用、aApp 调用、参数生成、结果总结、记忆压缩等。

所以从业务视角看，请求之间存在一个很自然的层级关系：

```text
Session：一个用户会话
  Message / Turn：用户发送的一轮消息
    Call：这一轮消息内部触发的具体调用
```

这个层级已经足够表达大多数后端逻辑：哪些请求属于同一个会话，哪些调用属于同一轮用户消息，以及某次具体调用是什么。

问题在于，当前系统里同时出现了 `session_key`、`lifecycle_id`、`trace_id`、`span_id`、`parent_span_id` 等字段。它们混合了业务模型和分布式链路追踪模型，导致后端理解成本变高。

> [!KEY]
> 当前真正需要先讲清楚的，不是字段名字本身，而是每个字段到底处在哪一层：Session、Message，还是 Call。

## 理想模型

理想情况下，我们应该先用业务概念表达这个系统，而不是直接引入 tracing 术语。

### 第一层：Session

`Session` 表示一个用户会话。

同一个会话里的多轮用户消息，都应该共享同一个会话标识。

这个字段的职责只有一个：回答“这些请求是不是属于同一个产品会话”。

理想字段可以叫：

```text
session_id
session_key
```

在当前系统里，对应字段是：

```text
x-remio-session-key
remio_session_key
```

如果后端只需要判断“多次请求是否属于同一个会话”，就应该只看这个字段。

### 第二层：Message / Turn

`Message` 或 `Turn` 表示用户发出的一轮消息。

一次用户消息可能触发多个内部调用，但这些调用都属于同一轮用户意图。

这个字段的职责是：回答“这些内部调用是不是由同一轮用户消息触发的”。

理想字段可以叫：

```text
message_id
message_request_id
turn_id
```

当前系统里，比较接近这个含义的是：

```text
x-remio-lifecycle-id
remio_lifecycle_id
```

但是 `lifecycle_id` 这个名字并不直观。它听起来像一个更大的生命周期，实际在主 Chat 链路里通常对应的是一轮用户消息的 `messageRequestID`。

因此，从业务语义看，它更应该被理解为：

```text
message_request_id / turn_id
```

### 第三层：Call

`Call` 表示某一轮用户消息内部的一次具体调用。

例如：

```text
用户问一个问题
  Agent 主模型调用
  RAG 检索后的总结调用
  工具参数生成调用
  aApp 结果反馈调用
  最终回答调用
```

这些调用都属于同一轮用户消息，但它们是不同的 call。

理想字段可以叫：

```text
call_id
parent_call_id
```

如果需要还原调用树，就需要 `parent_call_id`。如果后端只关心调用列表，不关心调用之间是谁触发谁，那么 `parent_call_id` 甚至可以不用。

当前系统里，对应的是 tracing 术语：

```text
x-remio-span-id
x-remio-parent-span-id
remio_span_id
remio_parent_span_id
```

也就是说，`span_id` 本质上就是一次 call 的 ID；`parent_span_id` 本质上就是父 call 的 ID。

## 当前系统实际在用什么

当前系统混合了两套表达方式。

一套是业务表达：

```text
session_key
lifecycle_id
```

另一套是链路追踪表达：

```text
trace_id
span_id
parent_span_id
```

当前字段可以这样理解：

```text
session_key
= Session 层，表示同一个产品会话。

lifecycle_id
= Message / Turn 层，表示一轮用户消息。但命名不够直观。

trace_id
= tracing 里的整条链路 ID。理论上也对应一轮用户消息触发的完整链路。

span_id
= tracing 里的一个步骤 ID。在我们的业务里，它基本等价于一次具体 call 的 ID。

parent_span_id
= 当前 call 的父 call，用来还原调用树。
```

这就产生了一个问题：

```text
Session -> Message -> Call
```

本来是三层业务结构，但当前字段看起来像：

```text
Session -> Lifecycle -> Trace -> Span
```

这会让人误以为系统里有四层甚至更多层概念。

实际并不是。

`trace_id` 和 `span_id` 是观测系统里的语言，不是业务系统必须理解的语言。

## 当前表达和理想表达的差异

### 差异一：`lifecycle_id` 和 `trace_id` 容易重叠

从合理语义看，`trace_id` 应该表示“一轮用户消息触发的完整内部链路”。

但当前 `lifecycle_id` 在主 Chat 链路里也基本表示“一轮用户消息”。

于是两者在业务理解上高度重叠：

```text
lifecycle_id：这一轮用户消息
trace_id：这一轮用户消息触发的链路
```

如果只是做业务归属判断，后端不应该同时依赖这两个字段。

更清晰的方式是：

```text
message_request_id / turn_id
= 业务层的一轮用户消息 ID

trace_id
= 观测层的链路 ID，只用于排障和 tracing
```

### 差异二：`span_id` 其实是 `call_id`

`span_id` 是 tracing 术语。对于业务后端来说，它不直观。

在我们的场景里，它表示的是：

```text
某一轮用户消息内部的一次具体调用
```

所以如果后端不是在做分布式 tracing，而是在做 LLM 调用管理，`call_id` 会比 `span_id` 更容易理解。

### 差异三：`parent_span_id` 只有在需要调用树时才有价值

如果后端只需要知道：

```text
这个 session 下有哪些 message
这个 message 下有哪些 call
```

那么 `parent_span_id` 不是必要字段。

它只有在要回答下面这种问题时才有价值：

```text
这个 RAG call 是被哪个主 Agent call 触发的？
这个工具调用是哪个模型调用产生的？
一次失败是从哪条子调用链传播出来的？
```

如果后端不需要调用树，只需要归属关系，那么：

```text
session_key + message_request_id + call_id
```

就已经足够。

### 差异四：aApp 和 Chat 不应该改变会话身份

当前某些路径里，`session_key` 可能被拼成：

```text
chat:<sessionId>
aapp:<aappId>:session:<sessionId>
```

但从产品视角看，aApp 只是一次 Chat 会话中的能力或工具，不一定是新的会话边界。

一个用户可以在同一个 Chat 里调起一个或多个 aApp。此时 aApp 信息应该出现在：

```text
feature
surface
entities.aappId
```

而不应该改变 session identity。

也就是说：

```text
session_key
```

应该只表达“属于哪个会话”，不应该同时编码“这次调用来自 Chat 还是 aApp”。

## 建议的后端理解方式

如果后端当前只是要处理“同一个会话里的多轮请求”，建议按下面的方式理解字段。

### 判断是否属于同一个会话

使用：

```text
x-remio-session-key
```

如果 header 缺失，再 fallback 到：

```text
remio_session_key
```

不要使用：

```text
trace_id
span_id
lifecycle_id
```

来判断会话归属。

### 判断是否属于同一轮用户消息

使用：

```text
x-remio-lifecycle-id
remio_lifecycle_id
```

但在概念上，应把它理解成：

```text
message_request_id / turn_id
```

而不是一个泛泛的 lifetime。

### 判断是哪一次具体内部调用

使用：

```text
x-remio-span-id
remio_span_id
```

但在业务上可以把它理解成：

```text
call_id
```

### 需要调用树时再看父子关系

使用：

```text
x-remio-parent-span-id
remio_parent_span_id
```

如果不需要调用树，这个字段可以忽略。

### `trace_id` 的位置

`trace_id` 更适合保留给观测系统。

它可以用于日志聚合、链路追踪、排障，但不建议业务逻辑依赖它来表达 Session 或 Message。

如果后端已有 `lifecycle_id` 表示一轮用户消息，那么 `trace_id` 不应该再被解释成另一个业务层级。

## 建议的概念命名

为了降低理解成本，可以把当前字段映射成下面这套业务语言：

```text
当前字段                         建议理解

session_key                      session_key
lifecycle_id                     message_request_id / turn_id
trace_id                         observability_trace_id
span_id                          call_id
parent_span_id                   parent_call_id
feature / surface / area          capability / entry / scenario
entities.aappId                  involved_aapp_id
```

这套映射的核心是：

```text
业务归业务，观测归观测。
```

业务后端只需要先理解：

```text
Session -> Message -> Call
```

tracing 字段只是为了排障时能把内部调用串起来，不应该反过来主导业务模型。

## 结论

当前系统并不是天然需要这么多业务概念。

从产品和后端业务视角看，最核心的结构只有三层：

```text
Session：一个会话
Message / Turn：一轮用户消息
Call：这一轮消息内部的一次具体调用
```

对应到字段上，最清楚的表达应该是：

```text
session_key
message_request_id / turn_id
call_id
parent_call_id（可选）
```

当前的 `trace_id / span_id / parent_span_id` 来自分布式 tracing 体系，其中 `span_id` 和 `parent_span_id` 对调用树有价值，但它们不应该成为业务后端理解会话和消息的主要入口。

最需要避免的是把当前字段误读成：

```text
Session -> Lifecycle -> Trace -> Span
```

更合理的理解应该是：

```text
Session -> Message / Turn -> Call
```

其中：

```text
session_key      负责 Session
lifecycle_id     当前承担 Message / Turn，但名字不理想
span_id          当前承担 Call
parent_span_id   当前承担调用树父子关系
trace_id         保留给观测和排障，不进入核心业务判断
```
