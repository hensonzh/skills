---
name: context-management-architecture
description: 设计或审查智能体上下文管理，划分模型输入、会话状态、外部事实、记忆投影与前端结果。
---

# Context Management Architecture

## 执行规则

- 先读现有请求组装、历史、状态与工具链路；只为本次涉及的数据明确归属和流转，不强制固定层数或独立服务。
- 分开可信指令与输入数据；检索、工具结果和抽取记忆保留来源，不因进入摘要或存储而获得指令权限。
- 每次模型调用按任务和预算选择历史、事实及记忆投影，避免重复注入；稳定前缀与动态信息分开，易变事实按需刷新。
- 压缩时保留目标、约束、进度和必要引用；已读取 ID 不代表模型仍可见，需要时重新加载，动作授权与执行状态从应用状态读取。
- 按恢复要求选择持久化边界，不能默认只在 run 结束落库；涉及副作用时定义幂等与结果查询，响应丢失不能直接当作未执行。
- 待确认动作绑定对象、参数版本和授权范围；已有授权的动作不重复确认，执行前重新校验权限与有效性。
- 产物和动作按复用、恢复需求显式存储；前端使用结构化状态，不从回答文本猜执行结果，也不把无关 UI 事件回灌模型。

设计输出相关决策和验证方式；审查先列问题，再给最小改法，不强制完整 schema 或报告模板。

## 按需参考

- 缺少关键需求：[discovery-checklist.md](references/discovery-checklist.md)。
- 定义数据结构：[schemas.md](references/schemas.md)。
- 分层、缓存和恢复细节：[context-management-reference.md](references/context-management-reference.md)。
- 需要完整方案样例：[worked-example.md](references/worked-example.md)。

完整记忆系统使用 `agent-memory-design`；前端流式体验使用 `agent-loop-streaming`，本 Skill 负责其上下文边界。
