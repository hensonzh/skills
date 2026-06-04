# 前端体验与 Work Panel 设计

这份 reference 帮设计者决定“用户应该看到什么”，再帮 coding agent 把这些决策落成组件和状态。不要一开始就陷入 SSE、WebSocket 或 provider event 细节。

## 三层语言

很多设计者只知道“不想让用户一直看 spinner”。把这个印象拆成三层：

1. **用户感知层**：用户看到“正在查找资料”“正在准备草稿”“请确认后提交”。
2. **产品决策层**：哪些过程可见、哪些必须确认、哪些只记录日志。
3. **工程实现层**：coding agent 实现事件、组件、状态机和 artifact schema。

回答或实现时按这个顺序推进，不要跳过体验决策直接写协议。

## 先定义不确定性

Agent loop streaming 的价值是降低等待时的不确定性。先回答：

- 用户提交后最担心什么？没响应、做错方向、泄露隐私、执行副作用、等太久？
- 用户需要知道 agent 正在做哪类工作？理解、检索、分析、生成、执行、等待确认？
- 用户需要中途介入吗？补充信息、确认写入、选择方案、取消？
- 最终结果是文本，还是可操作 artifact？
- 失败时用户最需要什么下一步？

如果这些问题没想清楚，流式 UI 很容易变成一堆“正在运行”的噪音。

## 五个设计判断

| 判断 | 设计问题 | 典型 UI 决策 |
| --- | --- | --- |
| 任务是否简单 | 是否只是短问答？ | 只用 streaming text，不上 work panel |
| 是否使用外部信息 | 是否查记录、知识库、网页？ | 显示“正在查找/读取”，最终展示来源摘要 |
| 是否生成结构化结果 | 是否有表单、报告、计划、草稿？ | artifact 独立区域展示 |
| 是否有副作用 | 是否发送、提交、修改、删除？ | preview + confirmation |
| 是否可能很久 | 是否超过几秒或后台任务？ | 可折叠进度、取消、稍后查看 |

命中越多，透明度越高。

## 四区布局

多数智能体前端可以用四区：

### Chat transcript

显示用户消息、assistant 简短确认、最终解释和下一步。不适合塞入工具日志、长检索列表、原始 JSON 或中间草稿。

### Work panel

承载可折叠过程，例如理解需求、查找资料、读取数据、生成 artifact、工具成功或失败。原则是“可见但不抢戏”，默认展示短状态，复杂细节放到展开态。

### Artifact area

承载结构化结果，例如表单、卡片、报告、表格、图表、计划、草稿、工单、可下载文件。如果 artifact 是用户真正要的东西，它应该比过程日志更醒目。

### Action area

承载确认、编辑、重试、取消、下载、提交、继续生成、联系人工。涉及副作用的动作必须先 preview，再 confirmation。

## 透明度等级

- **轻量透明**：适合简单问答。只显示 assistant streaming text 和轻量 loading。
- **标准透明**：适合多数工具型 agent。显示 2-5 个高层步骤、artifact、最终总结。
- **高透明**：适合高风险、高成本、长流程或 B2B 工具。work panel 可展开，显示来源、检查项、预览状态；写入/提交/发送前必须确认。

选择透明度时，优先考虑用户风险，而不是工程复杂度。

## 常见体验模式

### 普通问答

```text
用户发送
assistant 立即出现轻量占位
文本 delta 持续进入 assistant bubble
完成后停止 typing
```

不要为了简单问答强行展示复杂 work panel。

### 检索增强回答

```text
正在理解你的问题
正在查找相关资料
正在整理答案
最终回答 + 引用/来源摘要
```

来源不要作为长日志刷屏。优先在最终答案下方显示来源摘要。

### 工具执行型任务

```text
正在读取你的数据
正在计算/检查
正在准备结果
artifact 或最终建议
```

工具名要翻译成用户语言。`query_customer_orders` 应显示为“正在查找相关订单”。

### 生成 Artifact

```text
正在收集必要信息
正在生成草稿
artifact card 出现
assistant 总结如何使用
action area 提供编辑/确认/下载
```

Artifact 出现后，不要再让长文本把它顶走。

### 需要确认的副作用

```text
正在准备预览
展示 preview artifact
明确标注“尚未提交”
用户确认
显示提交中
显示提交结果和恢复入口
```

不要让 agent 在用户没确认前发送邮件、提交表单、修改数据、下单、删除记录或通知他人。

## 状态文案

好文案要短、具体、面向用户结果，不暴露内部实现。

| 工程状态 | 用户文案 |
| --- | --- |
| `requesting_model` | 正在理解你的需求 |
| `retrieval_started` | 正在查找相关资料 |
| `tool_call_start: orders.search` | 正在查找相关订单 |
| `tool_call_start: draft.email` | 正在准备邮件草稿 |
| `artifact_ready` | 草稿已准备好 |
| `confirmation_required` | 请确认后再提交 |
| `run_error` | 处理失败，可以重试 |

避免：“正在调用函数”“运行 tool”“等待 token”“执行 mutation”“AI 正在深度思考”但其实只是网络慢。

## Provisional Text 迁移

真实 agent loop 中，text delta 刚出现时，后面可能还会出现 function/tool call。为了保留流式体验，可以先展示，后归位：

```text
收到 text delta
  -> 先展示为 provisional assistant text

后续出现 tool/function call
  -> 把 provisional text 移入 work panel
  -> 折叠为 loop narration / intermediate text
  -> 清空当前 assistant bubble，等待最终回答

response completed 且没有 tool/function call
  -> 把 provisional text commit 为 final assistant bubble
```

实现注意：

- assistant bubble 支持 provisional/committed 状态。
- work panel 能接收 narration item。
- 工具事件到达时移动已显示文本，不复制两份。
- 最终答案只保留在 final assistant bubble；中间 narration 进入 work panel。
- 整个 loop 是否结束仍以 `RUN_FINISHED` 或项目等价事件为准。

## Tool Work Item 演进

工具调用不要刷成多条日志。一个工具调用应围绕同一个 `tool_call_id` 更新同一个 work item：

| 事件 | Work item 状态 | 用户文案示例 |
| --- | --- | --- |
| `TOOL_CALL_START` | running | 正在准备查询 |
| `TOOL_CALL_ARGS` | running | 查询条件已准备好 |
| `TOOL_CALL_END` | running | 正在执行查询 |
| `TOOL_CALL_RESULT` | completed/failed | 查询完成 / 查询失败 |

如果底层协议没有 `tool_call_id`，构造稳定 alias，例如 response id + output index、item id、tool execution id、parent message id + step index。

## 延迟空窗

真实 streaming 中常见两个空窗：

- thinking 消失后，到第一个 tool event 出现之间有延迟；
- assistant text 停止后，到下一次 tool event 或 final event 出现之间有延迟。

轻量优化：

- thinking completed 后，可以等下一条可见事件替换，避免 UI 闪空。
- text delta 停止 300-500ms 后，如果 run 还没结束，可以显示中性状态，如“正在准备下一步”。
- 下一条 text/tool/artifact/final/error event 到达时立即替换。
- 文案要克制，不要假装知道模型正在深度推理。

## Artifact 设计

Artifact 应解决“用户下一步怎么用”的问题。设计时定义：

- 类型：form、card、table、report、draft、plan、ticket、quote、file
- 关键字段
- 是否可编辑
- 是否需要确认
- 是否有副作用
- 是否可下载/分享
- 空状态和错误状态

只要用户需要点击、编辑、确认、下载或复用，就应该结构化，不要只做格式化文本。

## 移动端建议

- work panel 默认折叠，只展示当前最重要一步。
- artifact 使用全宽 card。
- action area 固定在 artifact 附近，不要藏到长文本底部。
- 长过程用“查看详情”展开。
- 错误和确认按钮保持可见。
