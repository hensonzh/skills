# Agent 上下文管理通用参考

这份参考用于设计、解释和评审智能体上下文管理方案。它不绑定某个模型厂商、框架或项目，重点解决一个问题：哪些信息应该在什么时候、以什么形式进入模型，哪些信息应该留在应用、工具、检索系统、前端或长期存储里。

## 目标

好的上下文架构应该同时满足：

- **可用**：模型能拿到完成当前任务所需的信息。
- **轻量**：不把无关历史、全量资料和大块工具 schema 塞进每轮请求。
- **新鲜**：易变事实通过读取工具或检索获得，减少依赖过期上下文。
- **可控**：副作用、权限、隐私、长期记忆和前端展示都有明确边界。
- **可恢复**：多轮对话、工具循环、确认动作和 artifact 能在刷新、重试、并发时保持一致。
- **可测试**：每层上下文都有可验证的输入、输出、过期和失败行为。

## 使用边界

本参考解决的是上下文编排问题：哪些信息进入模型，哪些信息留在应用、工具、检索、前端或长期存储里，以及它们如何通过 id、schema 和状态机衔接。

它不替代相邻系统的完整设计：

- 长期记忆的抽取、存储、检索、授权、删除和评估，需要独立细化。
- 前端 work panel、stream event contract、artifact 渲染和交互体验，需要独立细化。

当三者重叠时，context management 只定义边界：模型看到什么、应用保存什么、前端展示什么、哪些引用可以跨轮回传。

## 六层上下文模型

六层不是六段 prompt。只有当前任务必需的信息才进入模型上下文；其他层可以通过 history pointer、store ref、read tool、retrieval 或前端事件承接。

### 1. 稳定指令层

放置内容：

- agent 身份、目标、职责边界。
- 输出风格、语言策略、产品语气。
- 安全规则、拒答边界、升级策略。
- 工具使用原则、确认规则、隐私约束。
- 可用能力的轻量 manifest。

适合位置：

- system message、developer message、top-level instructions、agent config。

不要放：

- 当前时间、用户画像、业务记录、检索结果、上传文件内容、临时 UI 状态。

评审重点：

- 是否跨用户和跨轮次稳定。
- 是否短小、清晰、可缓存。
- 是否避免把业务数据当成规则硬编码。

### 2. 当前任务层

放置内容：

- 本轮用户消息。
- 当前页面、入口、任务类型、用户刚刚点击或提交的内容。
- 收到时间、locale、timezone、设备环境等当前 turn 需要的信息。
- 上传文件或图片的引用与用途说明。
- 用户已确认的表单数据或 action 参数。

适合位置：

- per-turn input、request context、current event payload。

不要放：

- 完整历史记录、全量 profile、长期偏好、大块知识库内容。

评审重点：

- 是否只描述当前 turn。
- 是否明确区分用户原话、系统包装的上下文、已确认数据和推断。
- 是否能在下一轮自然过期或由 history/session 承接。
- 是否为了缓存省略了本轮必须的新鲜信息，例如当前时间或确认 payload。

### 3. 会话历史层

放置内容：

- 用户和 assistant 的自然对话历史。
- 已返回给模型的工具结果。
- 已加载的参考资料摘要或正文。
- 多步骤 agent loop 中的 function call output。

实现方式：

- 模型平台管理的 thread/conversation/response chain。
- 应用侧消息表。
- rolling summary + recent messages。
- 混合方案：平台历史承接短期，应用侧持久化关键消息和 artifact。

评审重点：

- history 的来源是否单一可信，避免平台历史和应用侧重放互相重复。
- 超长对话如何截断或摘要。
- 旧工具结果是否仍然新鲜；易变事实应重新读取，而不是只靠历史复用。
- 工具结果是否需要进入模型历史，前端是否只展示脱敏版本。
- 并发 run 是否会覆盖同一 thread 的历史指针。

### 4. 运行状态层

放置内容：

- 当前 thread/run 的短期状态：loaded module ids、loaded reference ids、pending confirmation、last artifact id。
- 相对稳定环境快照的 hash，例如 locale/timezone/client 是否变化；不要用它替代 per-turn 时间。
- 最近少量客户端事件。
- 当前 workflow step、retry count、tool failure state。

适合位置：

- session store、run state、workflow state machine、server-side cache。

不要放：

- 大块原始资料、完整工具结果、敏感记录、长期用户记忆。

评审重点：

- 字段是否少而明确。
- 是否有 thread/run 作用域和过期策略。
- 刷新页面、重复提交、并发 run、工具失败时状态是否一致。

### 5. 外部事实层

放置内容：

- 用户资料、订单、票据、日程、设备状态、数据库记录。
- 文档、知识库、FAQ、产品手册、代码库、向量检索结果。
- 第三方系统返回的事实和结构化结果。

实现方式：

- read tools、retrieval、database query、file search、domain service。

评审重点：

- 是否按需读取，而不是默认注入。
- 是否有权限校验、租户隔离和最小字段返回。
- 是否带来源、时间戳和置信度。
- 是否处理空结果、过期数据、冲突事实和工具失败。

### 6. 长期记忆层

放置内容：

- 用户明确提供并长期有用的稳定事实。
- 偏好、习惯、默认设置、常用目标。
- 经过确认的个人化约束。
- 可复用但不确定的推断或会话摘要只应作为低置信假设处理，并由专门的长期记忆设计定义。

不要保存：

- 临时情绪、一次性任务细节、未确认推断、敏感数据、医疗/法律/财务高风险推断。

评审重点：

- 写入前是否有用户授权或清晰产品预期。
- 是否区分稳定事实、偏好、事件型摘要和推断型假设，且不会把推断当事实。
- 是否支持查看、更新、删除、过期和审计。
- 是否能解释来源，避免模型把推断当事实。

## 一次 Run 的推荐数据流

```text
user/app event
  -> normalize current input
  -> load session/run state
  -> build per-turn context
  -> select memory projections and external refs only when needed
  -> model call with stable instructions + current input + history pointer
  -> model asks for tool/retrieval when needed
  -> application executes tool with permissions and runtime inputs
  -> sanitized tool result goes to frontend if needed
  -> sanitized structured tool result goes back to model if needed
  -> model produces final answer or artifact
  -> update session state, artifact store, optional memory
```

关键点：

- 工具执行在应用侧，模型只提出调用意图和参数。
- 前端事件和模型输入可以使用不同 payload：前端看脱敏进度，模型看完成任务所需结构化结果。
- run 结束后再更新持久状态，或对每个副作用工具使用事务和幂等键。

## 信息放置决策表

| 信息类型 | 推荐位置 | 原因 |
| --- | --- | --- |
| agent 身份和安全规则 | 稳定指令层 | 跨轮稳定，可缓存 |
| 当前用户消息 | 当前任务层 | 只属于本轮 |
| 当前时间和 locale | 当前任务层 | 每轮可能变化，不应只为缓存省略 |
| 用户 profile | 外部事实层 read tool | 敏感且可能变化 |
| 已确认表单提交 | 当前任务层或业务状态 | 是当前动作的明确输入 |
| 工具结果 | 会话历史层，必要时 artifact store | 支撑后续推理和展示 |
| 大型文档 | 外部事实层 retrieval/file read | 体积大，需按需加载 |
| 已加载资料的短提示 | 运行状态层 | 避免重复检索，不重放全文 |
| pending confirmation | 运行状态层 | 阻塞副作用，需跨轮恢复 |
| 长期偏好 | 长期记忆层 | 跨会话复用 |
| 前端进度 | 前端事件 | 不应污染模型上下文 |
| 可编辑或可确认 artifact | Artifact store + 最小引用 | 前端渲染结构化结果，模型只看必要字段 |

## 事实新鲜度与优先级

同一事实可能出现在用户消息、历史、工具结果、profile、RAG、memory 里。设计时必须写清优先级：

- 当前用户明确输入优先于旧历史和 derived memory。
- 刚读取的业务 read tool 优先于历史里的旧工具结果。
- 带版本和发布时间的 RAG 结果优先于无版本摘要。
- 用户在 settings / UI 中确认的偏好优先于模型推断。
- 高风险领域中，未确认 derived memory 只能作为假设，不能当事实。

如果事实易变（订单状态、库存、价格、日程、权限、账户信息），默认通过 read tool 重新读取。history 的作用是提供对话连续性，不是事实缓存。

## Prompt 缓存和成本

降低成本的常见做法：

- 把稳定规则集中到固定 instructions。
- 避免在 stable instructions 中拼接时间、用户资料或检索结果。
- 不要为了保持 cache prefix 稳定而删除本轮必需信息；做法是稳定前缀缓存、动态信息后置。
- 不每轮重发完整历史，优先使用 provider-managed history 或摘要策略。
- 大量工具 schema 使用 manifest、namespace、deferred loading 或 tool search。
- reference 和知识库按需读取，并在 session state 记录短 id。
- 对长对话做 rolling summary，但保留最近若干轮原文。

风险：

- 过度摘要会丢失用户承诺、约束、否定信息和确认状态。
- 只依赖 provider-managed history 时，应用可能无法审计或恢复关键上下文。
- 只依赖应用侧重放时，容易重复注入工具结果并增加成本。
- 把当前时间、用户资料或检索结果混入 stable instructions 会让缓存和行为边界同时变差。

## 工具与副作用边界

工具分为四类：

- **read**：读取事实，无副作用。
- **evaluate/validate**：确定性评估或校验，无写入。
- **preview/plan**：生成候选方案或变更预览，不写入。
- **apply/mutate**：执行写入或外部副作用。

写类工具要求：

- 有结构化 confirmation，不只依赖一句自然语言。
- 有 action id、version、idempotency key。
- 能取消、过期、重试。
- 前端展示用户能理解的变更摘要。
- 工具结果区分用户可见摘要和内部细节。

## 前端事件和 Artifact

上下文架构要区分模型上下文和用户可见状态：

- 模型需要的是完成任务所需事实、工具结果和确认数据。
- 用户需要的是可理解的进度、可编辑 artifact、确认动作、错误恢复。

前端事件应该：

- 使用稳定 run id、message id、tool call id、artifact id。
- 不暴露原始 tool arguments、敏感 profile、完整检索文本或内部堆栈。
- 将 function/tool 名翻译成用户能理解的阶段。
- artifact 用结构化 schema 渲染，不从 assistant 文本里猜。
- provider 原始 stream event 应先转成应用侧稳定事件，再由前端消费。
- 后续 turn 需要引用 artifact 时，传 artifact id 和允许模型看到的字段；不要回灌完整 HTML、截图或前端内部状态。

## 常见反模式

- 把 prompt 当数据库，每轮塞入所有用户资料和历史记录。
- 把 session state 当长期记忆，导致重启或跨设备后丢失关键事实。
- 把长期记忆当垃圾桶，保存未确认推断和临时情绪。
- 工具结果既进入模型又完整推给前端，造成隐私泄露。
- 每轮重复加载相同文档，浪费成本并增加上下文冲突。
- 用自然语言确认执行高风险副作用，没有结构化 action id。
- 把 frontend event 全量注入 prompt，导致模型被 UI 噪音干扰。
- 在没有权限校验和字段最小化时开放 read tools。
- 为了 prompt cache 省略当前 turn 必需环境，导致模型使用旧时间或旧确认状态。
- provider history 和应用侧消息同时重放，导致工具结果重复、顺序错乱或 token 翻倍。

## 设计模板

整理一份 agent context 方案时，可以使用这个结构：

```markdown
# Context Architecture

## Goals
[agent 要完成的任务、上下文设计的目标]

## Context Layers
- Stable instructions:
- Per-turn context:
- Conversation history:
- Session/run state:
- Retrieval/tools:
- Long-term memory:
- Frontend events/artifacts:

## Data Flow
[一次 run 的输入、工具调用、结果回流、状态更新]

## State Model
[thread/run/user/action/artifact 的 id、生命周期和存储位置]

## Safety And Privacy
[权限、脱敏、确认、审计、删除]

## Failure And Recovery
[工具失败、并发、重复提交、上下文截断、memory 冲突]

## Test Plan
[关键单元测试、集成测试、回归场景]
```

## 现有项目审查模板

```markdown
# Context Architecture Review

## Findings
- Severity:
- Location:
- Issue:
- Impact:
- Recommendation:

## Current Context Sources
[用户输入、instructions、history、session state、tools、retrieval、memory、frontend events]

## Target Context Contract
[每层 owner、schema、进入模型/前端/持久化的字段]

## Migration Plan
[最小改动步骤、兼容策略、测试和观测]
```

## 测试建议

至少覆盖：

- 首轮和后续轮 request context 的差异。
- 同一 thread 下 history pointer 或消息重放是否正确。
- 长对话截断或摘要后是否保留确认状态和关键约束。
- 当前时间、timezone、locale、确认 payload 是否在需要时每轮新鲜。
- retrieval 空结果、冲突结果、过期结果和权限失败。
- 工具结果是否进入后续模型调用。
- 前端事件是否只展示脱敏内容。
- pending confirmation 刷新、重复提交、取消、过期和重试。
- 并发 run 是否产生状态覆盖。
- memory 写入、更新、删除和来源解释。
- prompt/cache 相关字段是否被动态信息污染。
