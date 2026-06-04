# 体验到事件的契约

这份 reference 把产品体验翻译成前端可实现的事件和状态。面向智能体设计者时，先确定“用户要看到什么”；面向 coding agent 时，再定义最少事件、状态机和组件更新规则。

核心原则：事件契约不是为了暴露 agent 内部细节，而是为了让前端稳定表达用户需要理解的 loop 阶段。

## 应用侧事件层

模型厂商的原始 streaming event 通常偏底层，例如 text delta、function call item、arguments done、response completed。它们适合后端驱动 loop，但不适合直接给前端展示。

前端真正关心的是：

- 智能体是否开始工作
- 是否正在 thinking/preparing
- 是否准备调用工具
- 工具参数是否准备好
- 工具是否正在执行
- 工具是否完成或失败
- 是否生成了结构化 UI
- 当前 run 是否彻底结束

推荐架构：

```text
model/provider raw stream
  -> backend agent loop parser
  -> application-side events
  -> SSE / WebSocket / AG-UI / framework stream
  -> frontend UI state
```

前端只消费稳定的应用侧协议，不直接依赖 OpenAI、Anthropic、LangGraph、Vercel AI SDK 或其他供应商的原始 stream event 名称。

## 先写 UI 状态

先列 UI 状态，再落事件：

| UI 需要展示什么 | 触发来源 | 结束条件 | 用户能做什么 |
| --- | --- | --- | --- |
| 正在理解需求 | run started | 第一个文本/tool/status 到达 | 等待或取消 |
| 正在查找资料 | retrieval/tool start | retrieval/tool result | 展开看来源 |
| 正在生成草稿 | artifact generation start | artifact ready | 等待 |
| 需要确认 | preview ready | 用户确认/取消 | 编辑、确认、取消 |
| 已完成 | final answer/artifact ready | run finished | 下载、继续、重试 |
| 失败 | run error/tool error | 用户重试/修改 | 重试、复制错误、联系人工 |

把这张表交给 coding agent 时，要求它实现成 run-level state、work-step collection、artifact collection、confirmation state 和 error state。不要把 provider stream event 原样渲染到聊天里。

## 推荐事件模型

对有工具调用的 agent，推荐这些应用侧事件或项目等价事件：

```text
RUN_STARTED               # 用户请求开始处理
TEXT_MESSAGE_CONTENT      # assistant 文本增量输出
TEXT_MESSAGE_END          # 当前文本段输出结束
TOOL_CALL_START           # 模型决定调用某个工具，工具调用开始出现
TOOL_CALL_ARGS            # 工具参数已经完整准备好
TOOL_CALL_END             # 模型的 function_call block 已结束，即将进入工具执行
TOOL_CALL_RESULT          # 工具执行完成，返回结果
ARTIFACT_CREATED          # 可选：结构化 UI 已生成
CONFIRMATION_REQUIRED     # 可选：需要用户确认副作用
RUN_FINISHED              # 整个 agent loop 结束，不会再继续调用工具
RUN_ERROR                 # stream 已开始后发生失败
```

UI 可以把它们翻译成：

| 应用侧事件 | 用户可见状态 |
| --- | --- |
| `RUN_STARTED` | 正在处理 |
| `TEXT_MESSAGE_CONTENT` | assistant 正在输出 |
| `TEXT_MESSAGE_END` | 当前文本段结束，等待下一步 |
| `TOOL_CALL_START` | 正在准备工具调用 |
| `TOOL_CALL_ARGS` | 参数已准备好 |
| `TOOL_CALL_END` | 正在执行工具 |
| `TOOL_CALL_RESULT` | 工具执行完成 |
| `ARTIFACT_CREATED` | 结果已生成 |
| `CONFIRMATION_REQUIRED` | 请确认后继续 |
| `RUN_FINISHED` | 已完成 |
| `RUN_ERROR` | 处理失败，可以重试 |

如果项目已有 AG-UI、Vercel AI SDK、LangGraph 或自定义协议，不要强行替换。把已有事件映射到同样的 UI 含义即可。

## Tool Call Work Item

一个工具调用在 work panel 里只对应一个 work item，并随事件演进：

```text
TOOL_CALL_START  -> running: 正在准备工具调用
TOOL_CALL_ARGS   -> running: 参数已准备好
TOOL_CALL_END    -> running: 正在执行工具
TOOL_CALL_RESULT -> completed/failed: 工具执行完成或失败
```

实现要求：

- 所有阶段都带同一个 `tool_call_id`，或能映射到同一个稳定 alias。
- 前端按 `tool_call_id` 更新同一个 work item。
- 不要为 start、args、end、result 各新增一条聊天消息。
- `TOOL_CALL_RESULT` 可以携带安全 artifact payload，用于渲染结构化 UI。

如果底层协议没有 `tool_call_id`，构造稳定 alias，例如 response id + output index、item id、tool execution id、parent message id + step index。

## 中间文本与最终文本

一个 model response 内可能既有 assistant text，也有 function/tool call。区分规则：

- 如果某一轮 model response 包含 function/tool call，这一轮里的 assistant text 是 loop narration / intermediate text。
- 如果某一轮 model response 没有 function/tool call，并且 run 结束，这一轮里的 assistant text 才是 final assistant response。

工程上可以采用 provisional 策略：

1. text delta 刚出现时，前端先临时展示为 provisional assistant text。
2. 如果后续出现 function/tool call，把这段 text 移入 work panel。
3. 如果 response completed 且没有 function/tool call，把这段 text commit 为 final assistant bubble。
4. `RUN_FINISHED` 是判断整个 agent loop 不会继续的终止信号。

## Payload 安全边界

发给前端的 payload 应遵守：

- 不发送完整 tool arguments，除非用户已经输入且需要回显。
- 不发送 secret、token、内部 prompt、隐藏 reasoning、完整数据库记录。
- 只发送 UI 渲染需要的安全摘要。
- 任何写入、发送、提交、删除都要有 preview 和 confirmation。

可以发送：用户可见 title/status、安全计数、来源摘要、artifact schema、错误摘要、side effect 是否已执行。

## 顺序规则

至少保证：

1. `RUN_STARTED` 后，输入区进入 busy。
2. 第一个可见反馈应尽快出现。
3. 文本 delta 可以和 work step 交替，但 UI 要区分 intermediate text 和 final answer。
4. artifact ready 后，应立即显示在 artifact area。
5. 需要确认时，停止自动执行，显示 action。
6. `RUN_FINISHED` 是成功流的最后事件。
7. `RUN_ERROR` 后必须恢复输入，并给出下一步。

## UI State 模型

推荐让 coding agent 维护类似状态：

```ts
type AgentLoopUiState = {
  runStatus: "idle" | "running" | "waiting_for_user" | "completed" | "failed";
  activeAssistantMessageId?: string;
  assistantTextById: Record<string, string>;
  workSteps: Array<{
    id: string;
    title: string;
    status: "queued" | "running" | "completed" | "failed" | "waiting_for_user";
    detail?: string;
  }>;
  artifacts: Array<{
    id: string;
    type: string;
    title?: string;
    status?: "preview" | "final" | "submitted" | "failed";
    data: unknown;
  }>;
  pendingConfirmation?: {
    id: string;
    title: string;
    description?: string;
  };
  error?: {
    message: string;
    retryable: boolean;
  };
};
```

这不是强制 schema，而是帮助代码代理保持边界清晰：assistant text、work steps、artifacts、actions 和 errors 是不同 UI 区域。
