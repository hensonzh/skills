---
name: agent-loop-streaming
description: 当智能体设计者、产品经理、独立开发者、vibe coder 或 coding agent 需要把“智能体 loop 流式输出”的模糊印象变成可设计、可实现、可验收的前端体验时使用；同时指导人类设计者做体验决策，并指导 Codex、Cursor Agent、Claude Code 等代码代理落地 agent 正在理解、检索、调用工具、生成 artifact、等待确认、完成或失败的 UI。
---

# Agent Loop Streaming

这个 skill 用来把“用户应该看到智能体正在工作”转译成可落地的前端体验。它同时服务两类对象：

- **智能体设计者**：产品经理、开发者、独立开发者、vibe coder。他们通常只有一个模糊印象，不清楚该展示哪些阶段、信息、动作和边界。
- **coding agent**：Codex、Cursor Agent、Claude Code、Replit Agent 等。它们需要把体验决策变成组件、事件、状态机、文案、错误处理和验收标准。

## 核心心智模型

模型的一次响应不一定等于最终回答。Agent loop 里，模型可能先输出 assistant text，再调用工具；工具结果回到模型后，模型继续生成下一轮响应。只有 loop 结束时，最后一轮不再调用工具的文本才是最终回答。

设计和实现时要区分：

- **intermediate text / loop narration**：中间说明，后续还可能调工具或继续工作。
- **final assistant response**：loop 结束前最后一轮的最终文本。
- **work state**：thinking、preparing、检索、工具执行、生成等过程状态。
- **structured artifact**：表单、卡片、报告、清单、计划表等由工具结果驱动的 UI。

前端不应直接消费模型厂商的原始 streaming event。后端 agent loop 应把原始模型事件转换成稳定的应用侧事件，再通过 SSE、WebSocket、AG-UI 或框架 stream 推给前端。前端围绕 run、message、tool_call_id、artifact_id 更新 UI，并用 `RUN_FINISHED` 或项目等价事件判断 loop 是否彻底结束。

## 使用方式

触发这个 skill 后，先判断任务属于哪一类：

1. **产品体验设计**：输出 UX 方案、信息层级、交互流程、状态文案。
2. **实现规格整理**：输出 event-to-UI contract、组件状态、验收清单。
3. **代码实现/改造**：读取代码，按现有框架实现。
4. **问题诊断**：定位 spinner、重复工具行、中间文本误当最终答案、artifact 不渲染等问题。

如果用户没有要求直接写代码，默认先给可执行的产品和实现 brief。如果用户要求改造现有代码，先读代码，再按本 skill 设计后实现。

## 面向设计者

如果用户还在探索体验，先帮助他们做 7 个决策，不要直接抛协议和组件名：

1. **任务复杂度**：简单问答、检索、工具执行、artifact 生成，还是长任务？
2. **用户风险**：等待、错误、隐私、副作用、成本，哪个最需要解释？
3. **可见阶段**：用户应该看到 3-6 个什么阶段？
4. **最终产物**：文本、artifact、确认动作，还是多者组合？
5. **中间过程边界**：哪些过程可见，哪些只进日志？
6. **确认边界**：哪些动作必须 preview 和 confirmation？
7. **失败恢复**：失败后能重试、编辑、取消、联系人工，还是稍后继续？

输出要让设计者能判断体验是否可信、清晰、可操作。

## 面向 Coding Agent

如果用户要实现、改造或交给代码助手，规格至少包含：

- 页面区域和组件列表
- event-to-UI mapping
- 状态进入/退出条件和用户文案
- work step 如何按稳定 ID 合并
- intermediate text 如何迁移到 work panel
- artifact schema 和渲染位置
- confirmation/action 的触发和阻塞规则
- 安全脱敏规则
- 错误、空结果、超时、取消、重试
- 验收场景和测试建议

不要只说“优化 agent loop streaming”。要把代码代理需要写的 UI 状态和行为说清楚。

## 输出格式

常规回答优先包含：

1. **体验目标**：这个 loop UI 要降低什么不确定性。
2. **屏幕区域**：chat transcript、work panel、artifact area、action/error area。
3. **状态分层**：哪些状态显示给用户，哪些只写日志。
4. **流式节奏**：文本、work step、工具结果、artifact、最终答案的出现顺序。
5. **状态文案**：短、具体、面向用户价值，不暴露内部实现。
6. **实现 brief**：组件、事件、状态、验收标准。

如果信息不足，先给默认 MVP 假设，并标出后续需要确认的问题。

## 体验原则

- 展示用户关心的进度，不展示 provider event、token 数、原始 tool arguments。
- 提交后先给轻量反馈，再显示具体工作步骤。
- 中间过程进入 work panel，最终结论保留在 final assistant bubble 或 artifact 总结中。
- 工具调用翻译成用户语言，例如“正在查找相关订单”，不要显示内部函数名。
- artifact 是主要结果时，让表单、卡片、报告、计划表成为主角。
- thinking/preparing 可以缓冲体验，但不要用假状态制造信任。
- 工具进度围绕同一个 `tool_call_id` 更新同一个 work item。
- 结构化 UI 由安全的工具结果驱动，不从 assistant 文本里猜。
- 失败时说明发生了什么、保留了什么、用户下一步能做什么。

## Reference 分工

- `references/frontend-work-panel.md`：体验模式、信息层级、work panel、artifact、provisional text、延迟空窗。
- `references/event-contract.md`：应用侧事件层、推荐事件模型、tool_call_id 合并、reducer 状态。
- `references/debugging-and-tests.md`：vibe coding brief 模板、代码代理任务边界、验收清单、常见问题。
