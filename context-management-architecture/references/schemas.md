# Agent 上下文最小 Schema 集

这份参考给出智能体上下文模块的最小可用 schema 形状。它不是某个框架的固定接口，而是一组在多数 agent 项目中都会出现的结构。落地时按需裁剪字段，但建议保留 id、版本、时间戳、来源和状态字段。

约定：

- 所有时间使用 ISO-8601 UTC 字符串。
- 所有 id 使用稳定、可作为幂等键或外键的字符串。
- 含敏感数据的字段必须区分 model_payload（进入模型）与 frontend_payload（推到前端）。

## 1. Per-turn Request Envelope

每轮交互的统一入口 payload。后端基于它构造模型请求。

```json
{
  "turn_id": "turn_2026-05-17T12:00:00Z_a1b2",
  "thread_id": "thread_user_42",
  "user_id": "user_42",
  "tenant_id": "tenant_acme",
  "received_at": "2026-05-17T12:00:00Z",
  "user_message": "Where is my order?",
  "environment": {
    "locale": "en-US",
    "timezone": "America/Los_Angeles",
    "client": "web",
    "client_version": "2026.5.17"
  },
  "current_task": {
    "entry": "support_inbox",
    "page_ref": "/tickets/123",
    "task_type": "qa"
  },
  "attachments": [
    { "kind": "image", "ref": "file_xyz", "purpose": "device_photo" }
  ],
  "confirmed_inputs": null,
  "client_event_refs": ["evt_abc"]
}
```

字段说明：

- `turn_id`：本轮唯一 id，用于日志、追踪、回放。
- `thread_id`：跨轮共享的 session 维度。
- `received_at`：属于当前 turn。不要因为追求 prompt cache 而复用上一轮时间。
- `confirmed_inputs`：上一轮 artifact/表单提交后的结构化数据。仅当本轮承载“已确认动作”时填写；后端仍需到 action store 校验状态、权限、过期和幂等。
- `attachments` / `client_event_refs`：只放引用，不放原始内容。原始内容由对应 store 持有。

## 2. Session State

跨轮保留的应用侧状态。建议保持小、字段明确、可序列化。

```json
{
  "thread_id": "thread_user_42",
  "user_id": "user_42",
  "instructions_version": "agent.v3",
  "history_pointer": "resp_xyz789",
  "loaded_modules": ["billing_policy", "shipping_policy"],
  "loaded_reference_ids": ["doc_42:v3"],
  "pending_action_id": "act_refund_001",
  "last_artifact_id": "artifact_42",
  "recent_client_events": [
    { "event_id": "evt_abc", "summary": "user opened order #555", "occurred_at": "2026-05-17T11:59:30Z" }
  ],
  "environment_snapshot_hash": "sha256:client-env:7f31...",
  "context_summary_ref": null,
  "turn_count": 7,
  "updated_at": "2026-05-17T12:00:01Z"
}
```

字段说明：

- `instructions_version`：当前 thread 绑定的 instructions 版本。版本切换时按策略迁移或开新 thread。
- `history_pointer`：平台 managed history 指针，例如 response id 或 conversation id。应用侧 history 时改为最新 message id。
- `loaded_*`：仅记 id 或短摘要，不重放完整资料。
- `recent_client_events`：滑窗，最多保留 N 条；进入 prompt 的可以更少。
- `pending_action_id` / `last_artifact_id`：与 action store 和 artifact store 交叉引用。
- `environment_snapshot_hash`：只用于判断 locale、timezone、client 等相对稳定字段是否变化；当前时间、收到时间等仍属于 per-turn envelope。
- `context_summary_ref`：长对话摘要或压缩上下文的引用。摘要不应覆盖 action/artifact 的结构化状态。

## 3. Memory Projection

上下文模块只需要知道“有一条长期记忆可供读取或引用”，不应在这里定义完整长期记忆系统。完整 memory record 的类型体系、抽取、存储、检索、TTL、授权和删除策略应由独立的长期记忆设计产出。

这里保留一个最小投影，用于说明 per-turn context 或工具结果里如何引用长期记忆。

```json
{
  "id": "mem_user_42_lang",
  "user_id": "user_42",
  "tenant_id": "tenant_acme",
  "scope": "user",
  "memory_type": "preference",
  "key": "communication.language",
  "value": "en",
  "source": {
    "kind": "user_explicit",
    "turn_id": "turn_2026-05-10T09:00:00Z_qq",
    "evidence": "user said 'please reply in English'"
  },
  "confidence": 1.0,
  "created_at": "2026-05-10T09:00:01Z",
  "updated_at": "2026-05-10T09:00:01Z",
  "ttl_at": null,
  "version": 1,
  "status": "active"
}
```

字段说明：

- `memory_type`：由具体长期记忆系统定义；上下文层只把它当作不透明分类，例如 stable fact、preference、episodic summary、derived hypothesis 等。
- `scope`：`user` / `team` / `tenant`，决定隔离粒度。
- `status`：`active` / `superseded` / `deleted`。删除走软删，配合审计。
- `version`：同一 key 的更新版本，便于冲突解决与回滚。
- 对模型暴露时建议只暴露 `memory_type` + `key` + `value` + `source.kind` 的最小投影；任何 derived/hypothesis 类信息必须带来源和置信度，不得当确定事实注入。

## 4. Tool Result Envelope

工具调用的统一返回结构。区分模型可见与前端可见两份 payload。

```json
{
  "call_id": "call_2026_xyz",
  "tool_name": "order_get",
  "ok": true,
  "model_payload": {
    "order_id": "order_555",
    "status": "shipped",
    "ship_eta": "2026-05-19"
  },
  "frontend_payload": {
    "summary": "订单 #555 已发货，预计 5/19 送达",
    "card": {
      "card_type": "order_status_card",
      "schema_version": "v1",
      "data": { "order_id": "order_555", "status": "shipped" }
    }
  },
  "error": null,
  "latency_ms": 142,
  "executed_at": "2026-05-17T12:00:02Z",
  "side_effect": false,
  "idempotency_key": null
}
```

字段说明：

- 失败路径：`ok=false`，`error = { code, message, retryable, user_facing_message }`。
- `side_effect=true` 的工具必须填 `idempotency_key`，并绑定 action 流程。
- 大体积结果应改为 `model_payload = { artifact_ref: "artifact_xx" }` + 详细数据落 artifact store。
- `model_payload` 不等于完整内部返回。它应只包含模型继续完成任务需要的字段；前端 payload 再按用户可见性单独脱敏。

## 5. Artifact

可被前端渲染、可被后续轮引用的结构化产物。

```json
{
  "artifact_id": "artifact_42",
  "thread_id": "thread_user_42",
  "type": "refund_preview_card",
  "schema_version": "v1",
  "data": {
    "order_id": "order_555",
    "refund_amount": 19.99,
    "currency": "USD",
    "reason": "wrong_item"
  },
  "rendered_summary": "Refund preview: $19.99 for order #555",
  "produced_by": { "tool_call_id": "call_2026_xyz", "turn_id": "turn_..." },
  "supersedes": null,
  "created_at": "2026-05-17T12:00:03Z",
  "status": "active"
}
```

字段说明：

- `schema_version`：前端按版本选择渲染组件。
- `supersedes`：新版本 artifact 取代旧版本时填上一版 id，便于追溯。
- `status`：`active` / `superseded` / `consumed` / `cancelled`。
- 模型回看 artifact 时只看 `type` + `schema_version` + 必要 `data`，不重发完整 rendered HTML/PNG。

## 6. Action / Confirmation

带副作用动作的状态机记录。

```json
{
  "action_id": "act_refund_001",
  "thread_id": "thread_user_42",
  "user_id": "user_42",
  "type": "refund_apply",
  "version": 1,
  "params": {
    "order_id": "order_555",
    "amount": 19.99,
    "currency": "USD"
  },
  "preview_artifact_id": "artifact_42",
  "idempotency_key": "thread_user_42:order_555:refund:v1",
  "status": "pending",
  "created_at": "2026-05-17T12:00:03Z",
  "expires_at": "2026-05-17T12:10:00Z",
  "confirmed_by": null,
  "confirmed_at": null,
  "applied_at": null,
  "applied_result_ref": null,
  "failure": null
}
```

状态流转：

```text
pending -> confirmed -> applied
   |          |
   +-> cancelled
   |          |
   +-> expired
              +-> failed (retryable | fatal)
```

约束：

- `idempotency_key` 应包含 thread/user/business-key/version，重复提交无副作用。
- `version` 用于同一业务对象多次预览或参数变化时区分 action；参数变化必须新建 action 或递增版本。
- `expires_at` 到达后自动转 `expired`，session state 中清理 `pending_action_id`。
- `failure.retryable=true` 时允许相同 idempotency_key 重新 apply；否则要求新建 action。

## 7. Instructions Version Record

instructions 的版本与缓存策略。

```json
{
  "id": "agent.v3",
  "name": "support-agent",
  "version": "v3",
  "checksum": "sha256:...",
  "content_ref": "prompt_store/support-agent/v3.md",
  "prompt_cache_key": "support-agent.v3",
  "compatible_session_versions": ["agent.v2", "agent.v3"],
  "released_at": "2026-05-15T00:00:00Z",
  "deprecated_at": null
}
```

迁移策略建议：

- 旧 thread 命中 deprecated 版本时，要么继续使用旧版本到自然结束，要么开新 thread。
- 默认不在同一 thread 中途切换 instructions 版本，避免行为漂移；若业务强制迁移，需记录迁移事件、旧版本、新版本和兼容策略。
- prompt cache key 与 version 严格绑定，version 改动则 cache 自动失效。

## 8. 字段最小化与脱敏约定

- 模型可见 payload：只放完成任务必需字段。
- 前端可见 payload：再脱敏一次，去掉内部错误码、调试堆栈、敏感 PII。
- 日志：可保留更完整字段，但需符合留存与删除策略。
- 任何含 PII 或敏感业务数据的字段必须显式打 `pii: true` 或归入 sensitive 字段集，由序列化层统一处理。

## 9. id 规范建议

- `turn_id`、`call_id`、`action_id`、`artifact_id`、`mem_*` 等使用前缀化字符串，便于日志分流和误用检测。
- thread/user/tenant id 不直接拼进 prompt 显示，必要时 hash 或截断。
- 跨系统外键统一使用同一 id 体系，避免“business id vs internal id”双映射难追溯。

## 10. 与上下文层的对应关系

| Schema | 主要归属层 | 备注 |
| --- | --- | --- |
| Per-turn envelope | 当前任务层 | 应用侧每轮统一入口 |
| Session state | 运行状态层 | thread/run 级，过期可清理 |
| Memory projection | 长期记忆层 | 只描述上下文层如何引用记忆；完整记忆系统另行设计 |
| Tool result envelope | 会话历史层 + 前端事件 | 模型与前端 payload 分离 |
| Artifact | 会话历史层 + 前端事件 | 后续轮通过引用复用 |
| Action / Confirmation | 运行状态层 + 业务系统 | 阻塞副作用 |
| Instructions version | 稳定指令层 | 缓存与迁移核心 |

落地时建议把这些 schema 在代码层固化为类型定义（TypeScript interface、Pydantic model、JSON Schema 等），并配合契约测试。
