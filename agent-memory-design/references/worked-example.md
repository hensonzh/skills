# Worked Example: 个人学习助手 LinguaCoach 0→1

场景：一个长期陪学的语言学习助手 LinguaCoach。用户与它的关系跨月跨季，随用户进步演化。每次会话可能是 5 分钟问答，也可能是 30 分钟练习。这是一个对长期记忆要求很高、隐私敏感度中等的场景，能完整演示四种记忆类型的写读路径。

## 1. Discovery Brief（记忆相关）

```markdown
## Agent 概述
- 名称：LinguaCoach
- 用户：登录账号语言学习者
- 场景：词汇 / 语法练习、口语模拟、考试备战、个性化复习

## 关键决策
- 任务形态：多会话长期陪伴
- 历史承接：平台 thread 承接当次；跨会话靠记忆
- 记忆类型：fact + preference + episodic + derived 全用
- 记忆体量：单用户 ≤ 500 条（软限）
- 抽取模式：explicit + 异步 post-run
- 存储：KV（fact / preference / derived）+ 向量（episodic）+ audit log
- 注入预算：每轮 ≤ 600 tokens
- 隐私：默认 opt-out 总开关 + per-category 细控
- 合规：GDPR / 个保法删除支持

## 默认假设
- 单用户独立账号，不跨用户共享
- 不存储敏感推断（健康、考试焦虑）
- 不存储语音原始内容（仅文本摘要）

## 风险与悬而未决
- 跨语言学习（同时学多语）的 scope 设计
- 是否做语音内容记忆（隐私 + 体积）
```

## 2. 记忆类型清单

| key | type | scope | 来源 | 生命周期 | 注入策略 |
| --- | --- | --- | --- | --- | --- |
| `profile.native_language` | fact | user | 注册 / explicit | 永久，可改 | 核心低敏注入 |
| `profile.target_language` | fact | user | explicit（可多值） | 永久 | 核心低敏注入 |
| `goal.target_cefr` | fact | user | explicit | 永久，可改 | 任务相关注入 |
| `goal.exam_date` | fact | user | explicit | 自动到期清理 | 备考相关 query 注入 |
| `pref.study.preferred_time` | preference | user | explicit / settings | 永久，可改 | allowlist 注入 |
| `pref.study.session_minutes` | preference | user | explicit / settings | 永久，可改 | allowlist 注入 |
| `pref.style.modality` | preference | user | explicit (visual/audio) | 永久，可改 | allowlist 注入 |
| `episodic.lesson_summary` | episodic | user | post-run 自动摘要 | 180 天 + 衰减 | 语义检索 |
| `episodic.milestone` | episodic | user | 系统事件（千词成就等） | 永久 | 重要节点注入 |
| `derived.weak_topic.<topic>` | derived | user | post-run 推断 | 30 天，需新证据续命 | 备课 / 复习场景注入，标 hypothesis |
| `derived.error_pattern.<pattern>` | derived | user | post-run 推断 | 30 天 | 同上 |

TTL 选值说明：episodic 取 `storage-and-retrieval.md` 推荐区间 90~180 天的高位（学习场景需要较长记忆窗），derived 取 14~60 天的中位 30 天。其它产品按业务调整，不必照搬。

不写入：

- 用户当下情绪（"今天好累"）。
- 用户对 agent 的吐槽。
- 假设性表达（"如果我去西班牙……"）。
- 学习焦虑、考试压力、自我评价类推断。

## 3. 写路径

### 触发

- **Explicit**：用户在 settings 修改 / 用户说 "以后用 X" / 用户说 "忘掉 Y"。
- **Post-run**：每次 lesson 结束后 background job 处理。

### Post-run 抽取流程

```text
lesson 结束（run finished）
  -> queue extraction job (job_id, thread_id, user_id, last_run_id)
  -> job 加载最近 N 轮对话 + lesson activity log
  -> rule 预筛：识别明确 marker 与活动事件
  -> LLM 抽取（strict schema, allow empty）
  -> validator：schema、阈值、白名单/黑名单、与现有记忆比对
  -> writer：
      - 完全相同 -> 丢弃，更新 last_seen_at
      - 同 key 不同 value -> conflict resolver
      - 新 key -> 写入主表 + audit
      - 命中黑名单 -> 拒绝 + audit
  -> 异步通知 UI（可选）："新增 2 条学习记录"
```

### 抽取 prompt 节选

```text
你在维护 LinguaCoach 的用户长期记忆。
任务：从这段 lesson 对话和活动数据中提取应跨会话保留的稳定信息。

允许的 key 前缀：profile.、goal.、pref.、episodic.、derived.
允许的 type：fact、preference、episodic、derived

不允许：
- 推断用户情绪 / 健康 / 焦虑 / 自我评价
- 把"假设"或"提问"当事实
- evidence 留长段原文（≤ 200 字）

输出 JSON 数组，允许空数组。

candidates:
```

### 冲突解决示例

- 用户先说 "我目标是 B1"，几周后说 "我想冲 B2"
  → `goal.target_cefr` 旧值 `B1` 标 superseded，新值 `B2` 写入，version+1。
- 用户原本 `target_language=es`，新 lesson 中说 "我也开始学 fr"
  → `target_language` 累积，存为多值 `[es, fr]`，不覆盖。
- 模型推断 `derived.weak_topic.subjunctive_present` 与上次相同
  → 丢弃但更新 `last_seen_at` 与 evidence_refs，TTL 续期。

## 4. 读路径

### 场景 A：用户开始新一节课

```text
new lesson trigger
  -> retrieval query:
       fact + preference: allowlist（核心、低敏、当前课程相关）
       episodic.lesson_summary: top 5 by recency * relevance("当前学习目标")
       derived.weak_topic: top 3，confidence ≥ 0.6
       budget: 600 tokens
  -> injection block (structured snippet)
  -> 注入到 system message 或 per-turn context
```

### 场景 B：用户问 "我上次学到哪了？"

```text
intent: recall
  -> memory_search(type=episodic, query="user 上次活动", top_k=3)
  -> 拼自然语言摘要回复
  -> 不需要把全部记忆都注入
```

### 场景 C：备考冲刺

```text
trigger: goal.exam_date 临近 7 天
  -> retrieval: derived.weak_topic + recent episodic + goal
  -> agent 主动给复习计划
  -> 注入时给模型加一句 system 提示：
     "用户考试在 N 天后，优先选择 weak_topic 与高频错误。"
```

## 5. 关键 Schema 实例

### Memory record（fact）

```json
{
  "id": "mem_user_42_native_language",
  "user_id": "user_42",
  "scope": "user",
  "type": "fact",
  "key": "profile.native_language",
  "value": "en",
  "source": {
    "kind": "user_explicit",
    "turn_id": "turn_2026-04-01_xx",
    "evidence": "I'm a native English speaker."
  },
  "confidence": 1.0,
  "version": 1,
  "status": "active",
  "created_at": "2026-04-01T07:30:00Z",
  "updated_at": "2026-04-01T07:30:00Z",
  "ttl_at": null
}
```

### Memory record（preference）

```json
{
  "id": "mem_user_42_pref_time",
  "user_id": "user_42",
  "scope": "user",
  "type": "preference",
  "key": "pref.study.preferred_time",
  "value": "morning",
  "source": {
    "kind": "user_explicit",
    "turn_id": "turn_2026-04-01_yy",
    "evidence": "I usually study before breakfast."
  },
  "confidence": 1.0,
  "version": 2,
  "status": "active",
  "updated_at": "2026-04-15T07:00:00Z"
}
```

### Memory record（derived）

```json
{
  "id": "mem_user_42_weak_subjunctive",
  "user_id": "user_42",
  "scope": "user",
  "type": "derived",
  "key": "derived.weak_topic.es.subjunctive_present",
  "value": { "topic": "subjunctive_present", "accuracy": 0.55, "samples": 12 },
  "source": {
    "kind": "post_run_derived",
    "evidence_refs": ["lesson_2026-05-10", "lesson_2026-05-14"]
  },
  "confidence": 0.72,
  "ttl_at": "2026-06-13T00:00:00Z",
  "version": 3,
  "status": "active"
}
```

### Memory record（episodic）

```json
{
  "id": "mem_user_42_lesson_2026-05-14",
  "user_id": "user_42",
  "scope": "user",
  "type": "episodic",
  "key": "episodic.lesson_summary",
  "value": {
    "topic": "subjunctive_practice",
    "summary": "用户在 2026-05-14 练习了西语虚拟式 12 题，准确率 55%，重点错在第三人称变位。",
    "occurred_at": "2026-05-14"
  },
  "embedding_ref": "vec_abc",
  "confidence": null,
  "ttl_at": "2026-11-10T00:00:00Z",
  "status": "active"
}
```

### Retrieval query

```json
{
  "user_id": "user_42",
  "scope": "user",
  "intent": "lesson_prep",
  "type_filters": ["fact", "preference", "episodic", "derived"],
  "semantic_query": "Spanish weak topics for subjunctive practice",
  "top_k_per_type": { "episodic": 5, "derived": 3 },
  "token_budget": 600
}
```

### Injection block（structured）

```yaml
user_memory:
  - type: fact
    key: profile.native_language
    value: en
  - type: fact
    key: profile.target_language
    value: [es]
  - type: fact
    key: goal.target_cefr
    value: B2
  - type: preference
    key: pref.study.preferred_time
    value: morning
  - type: preference
    key: pref.style.modality
    value: visual
  - type: derived
    key: derived.weak_topic.es.subjunctive_present
    hypothesis: 用户在虚拟式上准确率约 55%
    confidence: 0.72
    last_seen: 2026-05-14
  - type: episodic
    topic: subjunctive_practice
    summary: 5/14 用户练习虚拟式 12 题，准确率 55%
    occurred_at: 2026-05-14
```

system 段落附加一句：
> user_memory 中的 derived 仅作假设；与用户当下表达冲突时优先信用户。

## 6. 验收测试

| 编号 | 场景 | 期望 |
| --- | --- | --- |
| M1 | 用户说"以后用 morning" | 一轮内写入 preference；下次新会话拉到并注入 |
| M2 | 用户说"我同时学 es 和 de" | `target_language` 多值，不覆盖；version+1 |
| M3 | LLM 想抽取"用户很焦虑" | validator 拒绝（黑名单），audit 记录，不写 |
| M4 | 用户更正"我母语是中文" | 旧 native_language=en 标 superseded；新 zh 生效 |
| M5 | 用户在管理面板删 derived 全部 | retrieval 不再返回；audit 保留；30 天后清理 |
| M6 | derived 30 天没新证据 | TTL 过期，status=superseded；不再注入 |
| M7 | 同时手机 + 网页修改 preferred_time | conflict 按时间取新；version 单调递增 |
| M8 | 注入 token 预算超限 | 先丢低分 episodic，再丢 derived；核心低敏 fact 优先保留 |
| M9 | 用户请求导出 | 全量 JSON 含 fact / preference / episodic / derived，不含他用户数据 |
| M10 | 用户 GDPR 硬删除 | 物理删除 + 向量索引清理；audit 仅留 metadata |
| M11 | 跨语言新增目标"我开始学 fr" | target_language 累积；不覆盖 es |
| M12 | 模型在没有相关记忆时回答 | 不引用任何 memory snippet；监控注入污染率 |

## 7. 监控指标

每天看：

- 写入速率 / 拒绝率（按 type、按原因）。
- 抽取拒绝率（黑名单 / schema fail / 阈值不足）。
- 平均注入 token；fact / preference / episodic / derived 各自命中率。
- 用户编辑/删除频次（按 type）。
- TTL 清理量。
- 跨 scope 越界尝试（应为 0；非 0 立即告警）。
- 用户反馈"这条不对"比例。

## 8. 1→N 演进路径

- 引入"周报式 episodic 聚合"，把多个 lesson_summary 合并为 weekly_summary，降低体量。
- 给 derived 加用户确认机制，把高价值 derived 升级为 preference（用户点确认后转 type）。
- 多语种独立 sub-scope（`target_language=es` / `=de` 各自一套 derived）。
- 引入"安静模式"开关：一段时间内不抽取记忆（用户敏感场景）。
- 接入语音活动数据前做新一轮隐私评估与用户授权流程。
- 当单用户记忆 > 500 条时分桶并做向量索引压缩。
