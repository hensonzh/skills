---
name: context-management-architecture
description: 设计、审查或重构智能体上下文管理架构时使用：快速产出模型输入、会话历史、运行状态、外部事实、长期记忆边界、工具结果、artifact/action、缓存成本、恢复与副作用控制方案。
---

# Context Management Architecture

目标：快速给出一份可实现的上下文工程方案，明确什么进模型、什么留在应用、什么时候读取外部事实、什么只给前端、什么必须确认后执行。

## 快速架构方案

默认拆成四条线：

- **Model context** = stable instructions + per-turn envelope + history pointer + selected tool/retrieval/memory payloads。
- **App state** = session state + artifact store + action store + audit/log refs。
- **External facts** = read tools / retrieval / database / domain service，按需读取，不常驻 prompt。
- **Frontend state** = 脱敏进度、artifact UI、确认、错误；只把必要引用或 confirmed inputs 回灌模型。

一次 run：`event -> envelope -> load session/history -> model -> tool/retrieval -> sanitized result -> model/frontend -> final/artifact/action -> update stores`。

## 上下文分层

| 层 | 放什么 | 关键边界 |
| --- | --- | --- |
| Stable instructions | 身份、目标、风格、安全、工具原则 | 必须稳定、可缓存 |
| Per-turn context | 用户消息、received_at、locale/timezone、页面、附件 ref、confirmed_inputs | 只描述本轮 |
| Conversation history | 对话、模型已见工具结果、必要参考 | 不当事实数据库 |
| Session/run state | history pointer、loaded ids、pending action、last artifact | 小、短期、可恢复 |
| External facts | profile、订单、库存、文档、知识库 | 按需读，带权限和新鲜度 |
| Memory projection | 稳定偏好/事实的最小投影、来源、置信度 | 不展开完整记忆系统 |
| Artifact/action | 结构化结果、预览、确认、幂等写操作 | 不从文本反解析状态 |
| Frontend events | 进度、卡片、确认、错误的脱敏 payload | 不把 UI 噪音进 prompt |

默认策略：稳定规则进 instructions；当前动作进 per-turn；大块知识和易变事实按需读；短期状态显式存；artifact/action 独立存；长期记忆只传最小投影。

## 设计步骤

1. 写 brief：任务、用户、入口、历史、工具/RAG、artifact、隐私、性能、默认假设。
2. 列信息源并定归属：进模型、进 session、进工具、进 artifact/action、进前端、或不保存。
3. 定 schema：`per_turn_envelope`、`session_state`、`tool_result`、`artifact`、`action`、`memory_projection`。
4. 画数据流：输入、模型、工具、结果回灌、最终输出、状态更新。
5. 设计写操作：`preview -> confirmation -> apply`，必须有 `action_id`、`version`、`idempotency_key`。
6. 补测试：刷新、重复提交、并发、工具失败、过期、权限失败、上下文截断、PII 泄露。

## 审查现有项目

先找入口：request builder、history/thread、session/run store、tool registry、retrieval、artifact/action store、frontend stream adapter。

重点查：动态信息污染 stable instructions、profile/RAG/tool JSON/history 重复注入、易变事实复用旧 history、session state 当长期记忆、写操作只靠自然语言确认、工具结果未脱敏、provider history 和应用侧历史重复回放。

## 输出顺序

按这个顺序输出：**Discovery Brief** -> **Architecture** -> **Context Layers** -> **Data Flow** -> **Schemas** -> **Safety And Recovery** -> **Tests And Observability**。

审查现有项目时：先列 findings，再给目标架构和迁移步骤。

## 红线

不把 prompt 当数据库；不把 session state 当长期记忆；不把用户资料、RAG、工具结果无条件塞进每轮请求；不把易变事实只存进 history 后反复复用；不让写操作只靠一句自然语言确认。

## References

需要细节时读 [discovery-checklist.md](references/discovery-checklist.md)、[schemas.md](references/schemas.md)、[context-management-reference.md](references/context-management-reference.md)、[worked-example.md](references/worked-example.md)。
