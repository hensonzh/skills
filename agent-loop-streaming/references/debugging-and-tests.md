# Vibe Coding Brief 与验收

这份 reference 帮智能体设计者把“我想让用户看到 agent 正在工作”这种模糊想法，变成可交付给代码助手的需求，并判断实现是否达标。

## 从模糊印象到可执行 Brief

如果用户只说“我想要 agent loop 流式输出”，先不要直接要求代码助手实现。先补齐这张转换表：

| 模糊印象 | 需要追问或假设 | Brief 中应落成 |
| --- | --- | --- |
| 让用户看到 agent 在工作 | agent 做哪些类型的工作？ | work step 列表和状态文案 |
| 像 ChatGPT/Cursor 一样流式 | 是文本流式、过程流式，还是两者都有？ | text delta + work panel |
| 工具调用可见 | 哪些工具对用户有意义？ | tool category 到用户文案的映射 |
| 生成结果卡片 | artifact 类型和操作是什么？ | artifact schema + renderer |
| 需要用户确认 | 哪些动作有副作用？ | confirmation flow |
| 出错可恢复 | 用户能怎么继续？ | retry/edit/cancel/handoff |

如果信息不足，可以先给“默认 MVP 假设”，同时标出后续需要确认的点。不要因为用户不懂术语就停住。

## Vibe Coding Brief 模板

把下面模板填完整，再交给 Cursor、Codex、Claude Code、Replit Agent 或其他代码助手。

```text
我要为一个 [产品类型] 做 agent loop streaming 前端体验。

目标用户：
- [谁在用]
- [他们发起什么任务]
- [等待时最担心什么]

智能体能力：
- 可以回答文本
- 可以检索资料/记录
- 可以调用这些工具：[列出工具类别，不一定是函数名]
- 可以生成这些 artifact：[表单/卡片/报告/表格/草稿/工单等]
- 哪些动作有副作用，需要确认：[发送/提交/修改/删除/下单等]

页面结构：
- Chat transcript 显示用户消息和最终 assistant answer
- Work panel 显示 agent 正在做的步骤，可折叠
- Artifact area 显示结构化结果
- Action area 显示确认、编辑、重试、下载等按钮

状态设计：
- 提交后立即显示：[文案]
- 检索时显示：[文案]
- 调工具时显示：[文案]
- 生成 artifact 时显示：[文案]
- 需要确认时显示：[文案]
- 完成时显示：[文案]
- 失败时显示：[文案 + 下一步]

实现要求：
- 文本增量流式显示
- 工具/检索进度进入 work panel，不混入最终答案
- 每个 work step 用稳定 id 合并，避免重复行
- 如果模型先输出文本、后调用工具，先显示为 provisional text，随后迁移到 work panel narration
- artifact 从结构化 payload 渲染，不从文本解析
- 不展示原始 tool arguments、隐私数据、token 或内部 prompt
- run 完成后恢复输入；失败后显示重试
- 移动端 work panel 默认折叠

验收标准：
- [列出下面验收清单中适用的项]
```

## 给 Coding Agent 的任务边界

把需求交给代码代理时，要明确它应该做什么和不应该做什么。

应该做：

- 复用现有设计系统和组件风格。
- 新增或改造最少必要组件。
- 把 run、text、work step、artifact、confirmation、error 分成不同 UI 状态。
- 用稳定 ID 合并同一个 work step。
- 给每个用户可见状态写短文案。
- 加入基本 loading、empty、error、retry 状态。
- 补充能验证关键状态的测试或手动验收说明。

不应该做：

- 把 provider event、tool JSON 或内部日志直接展示给用户。
- 为了“透明”展示所有步骤。
- 在未确认时执行发送、提交、修改、删除。
- 用无法解释的 spinner 代替所有状态。
- 为了做 streaming 重写整个应用架构，除非用户明确要求。

## PM 验收清单

不看代码也能验收：

- 用户发送后 0.5-1 秒内有可见反馈。
- 用户能看懂 agent 当前处于哪类工作阶段。
- 简单问答不会被复杂 work panel 打扰。
- 工具型任务能看到 2-5 个清晰步骤。
- 中间过程和最终答案不会混在一起。
- artifact 出现的位置稳定，并且可操作。
- 涉及副作用时，界面明确显示“尚未提交/需要确认”。
- 错误状态有下一步：重试、编辑输入、取消、联系人工。
- 移动端不会因为过程日志把主要结果挤出屏幕。
- UI 没有展示内部函数名、原始 JSON、隐私字段或 token。

## 工程验收清单

需要实现或评审代码时检查：

- stream parser 能处理 partial chunk。
- 每个 event 有明确 UI 映射。
- work step 使用稳定 ID 合并。
- tool start、tool result 不会生成重复行。
- final answer 和 provisional narration 分离。
- `RUN_FINISHED` 或项目等价事件是 agent loop 的成功终止信号。
- text delta 停顿后的 preparing 状态会被下一条可见事件替换。
- artifact schema 有版本或类型字段。
- run success 是 terminal state。
- run error 会恢复输入控件。
- timeout、取消、重试有状态处理。
- SSE/WebSocket/框架 stream 多入口时，UI 收到的逻辑事件等价。

## 常见问题诊断

### 页面一直转圈

通常是因为只有 loading，没有阶段事件。

改法：

- 提交后立即显示轻量状态。
- 超过短阈值后显示 work panel。
- 后端或 mock stream 至少发出 work step。
- 长任务给取消或后台继续入口。

### 工具日志刷屏

通常是把内部事件直接展示给用户。

改法：

- 合并同一个 tool 的 start/result。
- 只显示用户可理解的高层步骤。
- 详细日志放到开发者 console 或展开态。
- 默认最多展示当前 3-5 个关键步骤。

### 用户分不清最终答案

通常是中间文本和最终文本混在同一个 bubble。

改法：

- 工具调用发生后，把之前文本移动到 work panel narration。
- 最后一轮模型文本才作为最终 assistant answer。
- Artifact 后方只保留总结和下一步。

### 工具状态跳成多条日志

通常是 `TOOL_CALL_START`、`TOOL_CALL_ARGS`、`TOOL_CALL_END`、`TOOL_CALL_RESULT` 被当成独立消息渲染。

改法：

- 为每次工具调用保留 `tool_call_id` 或稳定 alias。
- start/args/end/result 都更新同一条 work item。
- 只在 result 里展示完成或失败状态。
- artifact 从 result 渲染到 artifact 区，不塞进日志。

### Text 停住后界面空白

通常是 text delta 已停止，但下一步 tool event 或 final event 还没到。

改法：

- 300-500ms 后显示轻量 preparing 状态。
- 下一条可见事件到达时立即替换。
- run finished/error 时清除 preparing。
- 文案保持中性，不要假装知道模型正在深度推理。

### Artifact 被文本淹没

通常是 artifact 没有独立区域。

改法：

- 给 artifact 独立 card/section。
- assistant text 只解释 artifact。
- action button 放在 artifact 附近。
- 移动端 artifact 全宽展示。

### 状态看起来像假的

通常是前端 timer 伪装成 agent 状态。

改法：

- 把 timer 文案改成中性 loading，例如“仍在处理”。
- 只有真实事件到达时才显示“正在查找资料/正在生成草稿”。
- 无真实信号时不要写“正在推理”。

### 用户担心是否已经提交

通常是 preview 和 side effect 没分清。

改法：

- preview artifact 明确标“尚未提交”。
- 确认按钮文案写清动作，例如“确认并发送邮件”。
- 提交中、提交成功、提交失败分开显示。
- 成功后给可追踪信息：编号、时间、撤销/联系入口。

## 推荐演示场景

做 demo 或验收时，至少跑这些场景：

| 场景 | 预期 |
| --- | --- |
| 简单问答 | 只显示轻量 streaming，不出现复杂过程 |
| 检索回答 | 显示查找资料步骤，最终答案带来源摘要 |
| 工具调用 | work panel 显示清晰步骤且不重复 |
| 先文本后工具 | 前面的文本迁移到 work panel narration，最终答案单独显示 |
| 生成 artifact | artifact 独立出现，可操作 |
| 需要确认 | preview 和确认动作分离 |
| 工具失败 | 失败行可读，有重试或替代路径 |
| 空窗延迟 | thinking/preparing 不闪烁，下一条事件能替换 |
| 长等待 | 不空白，能取消或继续等待 |
| 移动端 | 结果优先，过程可折叠 |
