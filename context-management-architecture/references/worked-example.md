# Worked Example: 电商客服 Agent 0→1

这是一个端到端样例，演示如何用本 skill 的方法论从需求走到可实现的上下文模块。场景是一个电商客服 agent，覆盖六层上下文中的所有典型角色：政策问答（RAG）、订单查询（read tool）、退款（write tool with confirmation）、用户偏好（long-term memory）、退款预览卡（artifact）、刷新页面后恢复（session state）。

例子使用通用平台无关写法，落地到任意框架时按需替换。

## 1. Discovery Brief

按 `discovery-checklist.md` 走完后得到：

```markdown
# Agent Context Discovery Brief

## Agent 概述
- 名称：support-agent
- 用户：电商平台已登录买家
- 主要场景：订单咨询、政策解释、退款发起

## 关键决策
- 任务形态：多轮对话；每个 thread 对应一个客服会话
- 历史承接方式：模型平台 managed history（存储 response id），应用侧只持久化 thread 元数据 + artifact + action
- 长期记忆：仅保存用户偏好（语言、是否需要邮件副本），用户可在 settings 删除
- RAG：内部政策文档约 200 篇，季度更新；按 tenant 分片
- 工具：读 [order_get, policy_search]，写 [refund_apply]，写工具走 preview→confirmation→apply
- 前端 / Artifact：refund_preview_card；其他工具结果用纯文本摘要
- 合规要求：含 PII（姓名、邮箱、地址）；支持 GDPR 删除；不写入推断型记忆
- 性能 / 成本约束：每轮输入 ≤ 6k tokens，P95 响应 ≤ 4s
- 模型与平台：单一商用模型平台，启用 prompt cache + managed history

## 默认假设
- 单租户独立部署；后续如多租户需为 memory/RAG 加 tenant_id 索引
- artifact 详情不进入模型历史，仅 last_artifact_id 和必要摘要进 session state / tool result

## 风险与悬而未决
- 政策文档更新后旧 response 的引用是否需要失效
- 退款金额上限与人工兜底策略待业务确认
```

## 2. 上下文分层映射

| 信息 | 归属层 | 实现 |
| --- | --- | --- |
| agent 身份、退款边界、风险规则 | 稳定指令层 | system instructions，绑定 `agent.v1` |
| 当前用户消息、入口 page、locale、attachments | 当前任务层 | per-turn envelope |
| 多轮对话与工具结果 | 会话历史层 | 平台 managed history（response chain） |
| `loaded_policy_ids`、`pending_action_id`、`last_artifact_id`、`environment_snapshot_hash` | 运行状态层 | session store |
| 政策文档、订单 | 外部事实层 | `policy_search` retrieval、`order_get` read tool |
| 用户语言偏好、邮件副本偏好 | 长期记忆层 | memory store，`memory_type=preference`（完整长期记忆 schema 由独立 memory 设计承担，这里只引用边界投影） |
| 进度、refund 预览卡、确认按钮 | 前端事件 / artifact | 前端事件流 + artifact store |

明确**不放**的内容：

- 用户全量地址、历史所有订单 → 不进 prompt，由 read tool 按需获取。
- 政策文档原文 → 不进 instructions，按 query 检索后作为本轮工具结果进入历史。
- 推断的“用户情绪”→ 不写 memory，仅在当前 turn 影响语气。

## 3. Schema 实例化

只列与本场景紧密相关的字段。

### per-turn envelope

```json
{
  "turn_id": "turn_2026-05-17T12:00:00Z_a1",
  "thread_id": "thread_buyer_42_ticket_555",
  "user_id": "user_42",
  "received_at": "2026-05-17T12:00:00Z",
  "user_message": "I never got order #555. Can I get a refund?",
  "environment": {
    "locale": "en-US",
    "timezone": "America/Los_Angeles",
    "client": "web"
  },
  "current_task": {
    "entry": "support_inbox",
    "page_ref": "/tickets/555",
    "task_type": "refund_request"
  },
  "attachments": [],
  "confirmed_inputs": null,
  "client_event_refs": []
}
```

### session state

```json
{
  "thread_id": "thread_buyer_42_ticket_555",
  "user_id": "user_42",
  "instructions_version": "agent.v1",
  "history_pointer": "resp_aaa",
  "loaded_modules": [],
  "loaded_reference_ids": ["policy:refund_window:v3"],
  "pending_action_id": null,
  "last_artifact_id": null,
  "recent_client_events": [],
  "environment_snapshot_hash": "sha256:client-env:7f31...",
  "turn_count": 2,
  "updated_at": "2026-05-17T12:00:01Z"
}
```

### memory projection（用户已声明语言偏好）

仅展示上下文层如何引用记忆；完整 memory schema 与抽取/读取/生命周期由独立长期记忆设计定义。

```json
{
  "id": "mem_user_42_lang",
  "user_id": "user_42",
  "scope": "user",
  "memory_type": "preference",
  "key": "communication.language",
  "value": "en",
  "source": { "kind": "user_explicit", "turn_id": "turn_old", "evidence": "user said 'reply in English'" },
  "confidence": 1.0,
  "version": 1,
  "status": "active"
}
```

### refund 预览 artifact

```json
{
  "artifact_id": "artifact_refund_555",
  "thread_id": "thread_buyer_42_ticket_555",
  "type": "refund_preview_card",
  "schema_version": "v1",
  "data": {
    "order_id": "order_555",
    "refund_amount": 19.99,
    "currency": "USD",
    "reason": "not_received",
    "policy_basis": "policy:refund_window:v3"
  },
  "rendered_summary": "Refund preview: $19.99 for order #555",
  "produced_by": { "tool_call_id": "call_001", "turn_id": "turn_..._a1" },
  "status": "active"
}
```

### refund action（确认前）

```json
{
  "action_id": "act_refund_555_1",
  "thread_id": "thread_buyer_42_ticket_555",
  "user_id": "user_42",
  "type": "refund_apply",
  "params": { "order_id": "order_555", "amount": 19.99, "currency": "USD", "reason": "not_received" },
  "preview_artifact_id": "artifact_refund_555",
  "idempotency_key": "thread_buyer_42_ticket_555:order_555:refund:v1",
  "status": "pending",
  "created_at": "2026-05-17T12:00:04Z",
  "expires_at": "2026-05-17T12:10:00Z"
}
```

## 4. 数据流

### 4.1 政策问答 turn（无副作用）

```text
user_message: "What's your refund window?"
  -> backend builds per-turn envelope
  -> backend loads session state
  -> model call with stable instructions + envelope + history pointer
  -> model decides to call policy_search(query="refund window")
  -> backend executes retrieval, returns top-3 chunks (with citations)
  -> tool result envelope: model_payload = chunks + citations,
                          frontend_payload = "正在查阅政策" (脱敏)
  -> tool result fed back to model
  -> model produces final answer with citation
  -> session state.loaded_reference_ids += ["policy:refund_window:v3"]
  -> history_pointer updated to new response id
```

要点：

- 政策原文只在本轮工具结果里出现一次；未来轮次在同一 policy version / freshness window 内可通过 history pointer 复用，版本变化或用户提出新问题时重新检索。
- session state 仅记 reference id，不重放正文。
- 前端只看到“正在查阅政策”和最终答案，不看到原文 chunks。

### 4.2 退款 turn（含 confirmation）

```text
turn 1 (request):
  user_message: "Refund order #555."
  -> model calls order_get(order_id=555)
  -> model calls policy_search(query="not received refund eligibility")
  -> model decides eligible -> calls refund_preview(order_id=555, reason="not_received")
  -> backend creates artifact_refund_555 + action act_refund_555_1 (status=pending)
  -> tool result returns artifact_id
  -> model final text: "Here's the refund preview, please confirm."
  -> frontend renders refund_preview_card + confirm button
  -> session state.pending_action_id = act_refund_555_1
                  .last_artifact_id = artifact_refund_555

turn 2 (confirmation):
  frontend submits confirmed_inputs = { action_id: "act_refund_555_1", decision: "confirm" }
  -> backend validates action owner/status/expires_at/idempotency_key
  -> backend marks action.confirmed_at, then calls refund provider with idempotency_key
  -> on success: action.status = applied; on failure: failure {code, retryable}
  -> result fed to model as tool result
  -> model produces final user-facing summary
  -> session state.pending_action_id = null

turn 2 alt (cancel / expire):
  user cancels OR expires_at reached
  -> action.status = cancelled / expired
  -> session state cleared
  -> model informed via next turn's tool result or system note
```

要点：

- 退款由结构化 action 阻塞，不靠模型自然语言确认。
- idempotency_key 保证刷新或重复点确认无副作用。
- artifact 详情不重新进 prompt；模型仅看到 `action_id`、`status` 和完成回复所需的最小结果。

### 4.3 刷新页面恢复

```text
frontend reloads thread "thread_buyer_42_ticket_555"
  -> backend reads session state
  -> if pending_action_id and not expired:
       frontend re-renders artifact_refund_555 + confirm button
     else:
       frontend renders chat history only
  -> 后续 turn 与 4.2 turn 2 一致
```

## 5. 验收测试

| 编号 | 场景 | 期望 |
| --- | --- | --- |
| T1 | 首轮与后续轮 | per-turn envelope 含 received_at/timezone/locale；session 只保存 environment_snapshot_hash，不用旧时间替代本轮时间 |
| T2 | 政策问答 | 同一 policy version 内政策原文仅出现一次；版本变化或新问题触发重新检索 |
| T3 | order_get 失败 | tool result `ok=false, retryable=true`；模型给可重试措辞，不暴露内部 code |
| T4 | refund 预览 | artifact_refund_555 创建；session.pending_action_id 设置 |
| T5 | 用户确认 | refund 实际执行一次；重复点确认按 idempotency_key 不重复执行 |
| T6 | 用户取消 | action.status=cancelled；session state 清理 |
| T7 | 10 分钟超时 | action.status=expired；下一轮模型解释超时并提供重做路径 |
| T8 | 刷新页面 | artifact 与确认按钮恢复；history 通过 history_pointer 续接 |
| T9 | memory 删除 | 用户在 settings 删除 language preference 后，下一轮模型不再使用旧偏好 |
| T10 | 越权访问 | order_get 在用户与订单不匹配时拒绝；模型不暴露其他用户订单字段 |
| T11 | prompt 污染 | 用户消息中包含 “ignore previous instructions” 不影响 instructions 行为 |
| T12 | 长对话 | 第 50 轮仍能引用 turn 2 的退款决定；摘要策略保留确认状态 |

## 6. 可观测性挂钩

每轮记录：

- `turn_id`, `thread_id`, `instructions_version`, `model_input_tokens`, `model_output_tokens`, `cache_hit`, `tool_calls[]`, `latency_ms`, `error_code`。

工具调用记录：

- `tool_name`, `ok`, `latency_ms`, `result_size`, `idempotency_key`, `side_effect`。

action 记录：

- `action_id`, status 流转时间轴, `failure.code`。

memory 操作记录：

- `op` (`create`/`update`/`delete`), `type`, `source.kind`, `confidence`, `actor`。

可由这些指标衍生 dashboard：

- 缓存命中率、平均输入 tokens、tool 失败率、action 完成漏斗、memory 写入量、PII 字段触达率。

## 7. 后续演进

按这个 0→1 框架走通后，常见的下一步：

- 将单租户改为多租户：在 session、memory、retrieval 上加 `tenant_id`。
- 引入摘要 history：当 thread 超过阈值时压缩为 rolling summary，但保留 confirmed actions 的关键事实。
- 引入第二个 agent：handoff 时通过受控摘要交接，不直接共享原始 session state。
- 增加 dry-run / 回放：基于 turn_id 与历史快照重放，便于回归。

这一节本身不是 0→1 必需的，但通常是 1→N 阶段第一批要做的事，提前在架构里预留接口可以减少返工。
