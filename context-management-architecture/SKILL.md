---
name: context-management-architecture
description: 设计或审查智能体上下文管理，划分模型输入、会话状态、外部事实、记忆投影与前端结果。
---

# Context Management Architecture

## 执行步骤

1. 明确任务、信息源和现有调用链；改造时先读请求组装、历史、状态存储、工具与前端事件入口。
2. 为信息确定归属：稳定指令、本轮输入、对话历史、运行状态、外部事实、记忆投影、artifact/action 或前端事件。
3. 只为本次涉及的数据定义 schema，画清读取、注入、结果回传、状态更新与恢复路径。
4. 根据已有授权和操作风险设计写入：需要确认的动作使用预览和确认态；已授权的低风险动作可直接执行。可能重试的副作用定义幂等键，参数变化校验版本。
5. 验证受影响的截断、重复提交、并发、过期、权限和失败恢复场景。

## 分层规则

- 稳定指令保持稳定；当前时间、页面和用户输入放本轮上下文。
- 历史保留对话连续性；易变业务事实按权限和新鲜度要求重新读取。
- session/run 保存短期运行状态；长期记忆只注入当前任务需要的最小投影。
- 工具结果先筛选和脱敏；避免 profile、历史、检索和工具 JSON 重复注入。
- artifact/action 显式存储；前端消费结构化事件，不从回答文本反解析执行状态，也不把无关 UI 事件回灌模型。

设计时输出相关决策、数据流和验证方式；审查时先列问题，再给最小改造步骤。完整 schema 或迁移计划仅在任务需要时展开。

## 按需参考

- 缺少关键需求：[discovery-checklist.md](references/discovery-checklist.md)。
- 定义数据结构：[schemas.md](references/schemas.md)。
- 分层、缓存和恢复细节：[context-management-reference.md](references/context-management-reference.md)。
- 需要完整方案样例：[worked-example.md](references/worked-example.md)。

完整记忆系统使用 `agent-memory-design`；前端流式体验使用 `agent-loop-streaming`，本 Skill 负责其上下文边界。
