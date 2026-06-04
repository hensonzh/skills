# Storage and Retrieval

记忆系统的存储介质选型、检索路径与注入策略。读写路径要分别设计，不要把"全部塞进向量库"作为默认方案。

## 1. 介质选型决策表

| 类型 | 推荐介质 | 原因 |
| --- | --- | --- |
| fact | KV / 关系表（索引：scope+key） | 主要 key lookup；需要事务、版本、审计 |
| preference | KV / 关系表 | 同 fact，体量小，对一致性要求高 |
| episodic | 向量库（带元数据） | 语义检索为主；需要 recency 与衰减 |
| derived | KV + audit log | 需保留 evidence，可被 fact 覆盖 |
| 关系/图谱（用户-关系-实体） | 图数据库或多对多关系表 | 仅在产品有显式关系建模需求时使用 |

实际架构通常是**混合**：

- KV/关系表 存 `fact` / `preference` / `derived`。
- 向量库 存 `episodic` 摘要（不存原文）。
- audit log 表 存所有写入与删除。
- 可选：物化视图加速"按 user 列出全部记忆"等管理界面查询。

不要"只用一个向量库存所有"——会导致 fact 检索召回不稳定，hallucination 难定位。

## 2. Scope 与分片

- **scope**：`user` / `team` / `tenant` / `global`。
- 索引必须以 scope 为最高维度，所有 query **强制**带 scope filter。
- 跨 scope 共享必须在 schema 显式声明（如 `team_shared=true`），默认不分享。
- 多租户场景：`tenant_id` 作为存储分片键；用户 id 仅在 tenant 内唯一。
- 切换"代理人格"或角色（如医生模式 vs 朋友模式）必须切换或重置 scope，避免泄露。

## 3. 索引设计

### KV / 关系表

- 主键：(scope, scope_id, key, version)。
- 唯一索引：(scope, scope_id, key, status='active')，确保活跃版本唯一。
- 辅助索引：
  - (scope, scope_id, type)：按类型列表
  - (scope, scope_id, updated_at)：最近更新
  - (scope, scope_id, ttl_at)：过期清理
- status 字段（active / superseded / deleted）必须加索引。

### 向量库

- 向量本体 + 元数据：`type`、`scope`、`scope_id`、`occurred_at`、`topic_key`、`confidence`、`ttl_at`、`status`。
- 检索时强制带 scope filter，不要"先检索再过滤"。
- 索引建议为每个 scope 独立 namespace，便于配额与删除。

### Audit log

- 不可变追加表；每行：op（create / update / delete）、actor、reason、before、after、source、turn_id、created_at。
- 软删除时同样写一条；硬删除时清主表，audit 仅保留 metadata（按合规要求）。

## 4. TTL 与生命周期

| 类型 | 默认 TTL | 衰减策略 |
| --- | --- | --- |
| fact | 无 TTL | 不衰减；只在用户主动删除或显式更正时变化 |
| preference | 无 TTL | 不衰减；可被新值覆盖 |
| episodic | 90~180 天 | 检索打分时按 recency 衰减 |
| derived | 14~60 天 | 衰减；过期需新证据续命，否则降级 superseded |

`recency_decay = exp(-(now - last_seen) / τ)`，τ 按类型设置：

- episodic τ ≈ 30~60 天。
- derived τ ≈ 14 天。
- fact / preference 不参与衰减打分。

到期清理任务每天/每周跑一次：

- 把 `ttl_at < now` 的记录置为 `status=superseded` 或物理删除（按合规策略）。
- 同步清理向量索引。
- 写 audit。

## 5. 检索路径

### 直接 key 查找（fact / preference / 已知 key 的 derived）

- API：`memory_get(scope, scope_id, key)`。
- 不依赖语义匹配，精确返回当前 active 版本。
- 不命中返回空，不要 fallback 到语义检索。

### 主题 / 命名空间扫描

- API：`memory_list(scope, scope_id, type?, key_prefix?, limit?)`。
- 用于 prompt header 的 allowlist 注入（核心 fact + preference）或管理面板列表。

### 语义检索（episodic / 自由形式 derived）

- API：`memory_search(scope, scope_id, query_text, type_filter?, top_k)`。
- 必须经过 scope filter；返回带 evidence、occurred_at 与 score。
- 推荐返回 top_k 偏小（5~10），由排序与预算控制最终注入量。

### 混合检索（推荐默认顺序）

```text
1. fact + preference: memory_list allowlist（核心、低敏、当前任务相关）
2. derived: memory_list by key_prefix 或 memory_search by intent
3. episodic: memory_search by current intent + recency window
```

这种顺序符合"事实优先、推断保留、回忆兜底"原则，也便于在 token 预算内裁剪。

## 6. 排序

记忆排序得分一般写成：

```text
score = α * relevance
      + β * recency_decay
      + γ * confidence
      + δ * type_priority
```

经验权重（仅供起步，需 A/B 调）：

| 类型 | α relevance | β recency | γ confidence | δ type_prio |
| --- | --- | --- | --- | --- |
| fact | 0 | 0 | 0 | 1.0（最高优先级，仍受 allowlist / privacy / budget 约束） |
| preference | 0 | 0 | 0 | 1.0（最高优先级，仍受 allowlist / privacy / budget 约束） |
| episodic | 0.6 | 0.3 | 0.1 | 0.0 |
| derived | 0.3 | 0.2 | 0.5 | 0.0 |

不要拍脑袋固定权重，必须 A/B 才能定。监控注入命中率与污染率，再调权重。

## 7. 注入预算与格式

### 预算

- 给记忆设独立 token cap，建议 ≤ 当前轮总输入的 5~10%。
- 超预算时按"核心事实和偏好优先、推断/回忆后置"裁剪：
  - 只保留当前任务相关、低敏、allowlist 内的 fact / preference。
  - 先丢低分 episodic，再丢低 confidence derived；必要时再裁剪非核心 preference。
  - 任何裁剪都要记录到日志，便于评估"是否裁掉了关键信息"。

### 格式（推荐 structured snippet）

```yaml
user_memory:
  - type: fact
    key: profile.native_language
    value: en
  - type: preference
    key: pref.style.modality
    value: visual
  - type: derived
    key: derived.weak_topic.subjunctive
    hypothesis: 用户在虚拟式上准确率约 55%
    confidence: 0.72
    last_seen: 2026-05-14
  - type: episodic
    topic: subjunctive_practice
    summary: 5/14 用户练习虚拟式 12 题准确率 55%
    occurred_at: 2026-05-14
```

格式要点：

- structured 优先；natural language 仅在用户视图或前端展示用。
- derived 字段必须用 `hypothesis` 而不是 `value`，并附 confidence。
- 不把 evidence 原文塞进 prompt（隐私 + 噪声）。
- 配套在 system 段落中加一句指令：`"user_memory 中的 derived 仅作假设，与用户当下表达冲突时优先信用户。"`

### 反例

- 直接把 LLM 抽取的 evidence 原文塞 prompt。
- 不标 type / confidence → 模型把 derived 当 fact。
- 把 fact 和 episodic 混在一起，不区分注入优先级。
- 注入预算无上限 → 长尾用户上下文爆炸。

## 8. 读时脱敏

- PII 字段（精确地址、邮箱、电话）进 prompt 前必须按需要 mask。
- 高敏类目即使存了也默认不进 prompt，只在显式需要时通过受限 tool 读。
- 前端展示记忆时按渠道决定可见字段集；evidence 默认折叠为"展开来源"。
- 跨用户共享 scope（team / tenant）时，写入者标识默认匿名化。

## 9. Read-side Schema

### Retrieval Query

```json
{
  "query_id": "q_xxx",
  "scope": "user",
  "scope_id": "user_42",
  "intent": "lesson_prep",
  "type_filters": ["fact", "preference", "episodic", "derived"],
  "key_prefix": "study.",
  "semantic_query": "user weak Spanish topics",
  "top_k_per_type": { "episodic": 5, "derived": 3 },
  "token_budget": 600,
  "exclude_keys": []
}
```

### Injection Block（返回形状）

```json
{
  "snippets": [
    { "type": "fact", "key": "profile.native_language", "value": "en", "confidence": 1.0 },
    { "type": "preference", "key": "pref.study.preferred_time", "value": "morning", "confidence": 1.0 },
    { "type": "episodic", "topic": "subjunctive_practice", "summary": "...", "occurred_at": "2026-05-14", "score": 0.82 }
  ],
  "tokens_used": 320,
  "truncated": false,
  "dropped": []
}
```

把 retrieval 返回固化为单一 schema：前端展示、prompt 拼接、回放评估都用同一形状。这样后续做"为什么注入了这条 / 为什么没注入这条"的归因和回放才有抓手。

## 10. 操作经验

- 写读分库，避免读热点把写打挂。
- 向量索引重建是高成本操作，schema 升级时要灰度。
- 任何"重新摘要"或 batch 操作必须有版本号，避免与在线写入互相覆盖。
- 给 retrieval 写一个 dry-run 接口，测试时只返回 snippets 不写日志，便于调试 prompt。
- 单用户记忆条数过多（如 > 1k）时考虑分桶 + 子摘要，不要无限堆。
