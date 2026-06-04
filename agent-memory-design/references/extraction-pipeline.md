# Extraction Pipeline

负责回答"从会话里提炼什么、什么时候提炼、怎么提炼、提炼后是否真的写入"。

## 1. 记忆信号识别

### 应该被识别为可写入的信号

- **用户 explicit 声明**：`"I prefer X"`、`"我对 Y 过敏"`、`"以后请用 Z"`、`"Remember that…"`。
- **长期目标**：`"目标是 12 月通过 B2 考试"`。
- **稳定身份事实**：姓名、生日、所在地、母语、职业。
- **设置变更**：用户在 settings 页面操作。
- **已确认动作**：里程碑达成、退款理由、签到等结构化事件。
- **反馈与纠正**：`"其实我已经不在 New York 了"`、`"刚才那条记错了"`。
- **重复行为**：连续 N 次同样选择（谨慎，倾向 derived）。

### 不应该被写入的信号

- 一次性任务细节。
- 临时情绪、困惑、抱怨（`"今天好累"`、`"我搞不懂"`）。
- 模型从一句话推断的人格、状态、能力标签。
- 敏感推断：健康、财务、宗教、政治、性取向、未成年人相关。
- 用户假设性表达（`"如果……我就……"`）。
- 未确认的过往事件（`"我好像去过……"`）。
- 用户对 agent 的评价（除非作为 explicit 偏好）。

### 边界规则

- 高置信 + 明确表达 → 直接写。
- 高置信 + 隐含表达 → 写为 pending 并向用户确认。
- 低置信 → 不写，或写为 derived 并附 evidence。
- 任何敏感类目 → 默认不写，需走单独授权流程。

## 2. 触发模式

四种模式可以并存：

### 用户显式（Explicit）

- 用户主动说"记住"或在 settings 修改。
- 写入门槛最低，无需 LLM 抽取。
- UI 必须有 ack 反馈，否则用户没有"被听到"的感觉。

### 同步 in-loop

- agent loop 内识别到信号当轮抽取并写入。
- 优点：本轮就生效，可以在回复中确认。
- 缺点：增加每轮延迟和 token；容易抽取过度。
- 适合：高价值且容易识别的信号（如显式偏好、显式纠正）。

### 异步 post-run

- run 结束后入队 background job 抽取。
- 优点：不影响响应延迟，能利用更全的上下文。
- 缺点：不在当轮生效，下一次会话才能用。
- 适合：episodic 摘要、derived 推断。

### 离线 batch

- 定时（每天/每周）扫描历史窗口做提炼或聚合。
- 适合：跨多 session 才能识别的模式（错误规律、目标进度）。
- 必须有版本管控，避免覆盖近期人工/在线更新。

推荐组合：explicit + post-run 为基线，离线 batch 在体量大时补充。

## 3. 抽取方法

### 规则 / 模板

- 适合明显 marker：`"我喜欢/讨厌/过敏于/希望/打算"`。
- 优点：可解释、零成本、零延迟。
- 缺点：召回低、维护成本高、跨语言适配难。

### LLM 抽取

- 给定严格 schema，让 LLM 输出候选 memory record。
- 必须 strict schema + 允许返回空数组。
- prompt 必须强调"不确定不要编"。
- 默认温度 0 或近 0；同一段对话多次抽取应稳定。

### 混合（推荐）

- 规则做初筛（识别"是否值得抽取"）。
- LLM 做结构化（具体 type/key/value/evidence/confidence）。
- validator 做校验。

## 4. 抽取 Prompt 模板

```text
你正在为用户的 agent 维护长期记忆。
任务：从下面这段对话和事件数据中，提取**值得跨会话保留**的稳定信息。
仅输出结构化候选，不要输出叙述。

candidate schema:
{
  type: "fact" | "preference" | "episodic" | "derived",
  key: string,                  // dot.path 风格，前缀必须在白名单内
  value: any,                   // 与 type 一致
  evidence: string,             // 原文片段，≤ 200 字
  confidence: number            // 0..1
}

规则：
- 只提取用户**明确表达**的稳定信息或**已发生**的事件。
- 不要写入推断的情绪、健康、焦虑、能力评价。
- 不要把"假设"或"提问"当事实。
- 与已有记忆一致或冲突时，输出新值并标 supersedes_key。
- 矛盾、不确定、一次性任务细节 → 不输出。
- 输出空数组也是合法答案。

允许的 key 前缀：{whitelist}
禁止的 key 前缀：{denylist}

对话片段：
{messages}

candidates:
```

实践要点：

- 始终给 whitelist 与 denylist；不留通配 key。
- evidence 强制截断；不要把整段对话当证据。
- 复杂结构 value 做嵌套 schema，避免模型用自然语言塞 value。
- prompt 末尾留下"空数组合法"的明确出口，否则模型会强行填东西。

## 5. Extraction Job Schema

抽取任务作为独立工作单元，便于追踪、重试与回放。最小形状：

```json
{
  "job_id": "job_2026-05-17T12:00:00Z_xx",
  "scope": "user",
  "scope_id": "user_42",
  "trigger": "post_run",
  "source_window": {
    "thread_id": "thread_user_42",
    "from_turn_id": "turn_2026-05-17T11:00:00Z_aa",
    "to_turn_id": "turn_2026-05-17T11:55:00Z_zz",
    "run_ids": ["run_2026-05-17T11:55:00Z_q"]
  },
  "extractor_version": "extractor.v3",
  "status": "queued",
  "candidates_count": null,
  "accepted_count": null,
  "rejected_count": null,
  "error": null,
  "created_at": "2026-05-17T12:00:00Z",
  "started_at": null,
  "finished_at": null
}
```

字段约定：

- `trigger`：`explicit` / `sync_in_loop` / `post_run` / `offline_batch`，与第 2 节四种模式对齐。
- `extractor_version`：抽取规则 + LLM prompt + validator 的版本组合；变更必须升版以便回归。
- `status`：`queued` / `running` / `succeeded` / `failed` / `cancelled`。失败重试需新 job 而非原地修改，便于审计。
- `source_window`：用范围 id 而非完整对话内容；原文留在会话历史，避免 job 体积爆炸。
- 任何写入或拒绝都关联 `job_id`，写入 audit log，便于反向追溯抽取层质量。

## 6. 校验层（Validator）

LLM 输出后必须经过应用侧校验，至少包括：

- **schema 严格匹配**：字段、类型、长度、enum。
- **type-confidence 阈值**：例如 `fact` 至少 0.8、`derived` 至少 0.5。
- **key 命名**：dot.path、白名单前缀。
- **黑名单 key**：高敏类目直接拒绝并写日志。
- **evidence 长度**：超过 N 字截断或拒绝。
- **与现有记忆比对**：完全相同丢弃，相同 key 不同 value 走冲突解决。
- **频次限制**：单次 run 抽取上限（防止刷库）。
- **去 PII**：在 evidence 中扫描身份证、银行卡、电话等模式，命中则脱敏或拒绝。

校验失败的候选**必须**进 audit log，便于回溯抽取层质量。

## 7. 去重与冲突解决

### 完全重复

- key + value + scope 相同：丢弃，但更新 `last_seen_at` 与 hit count。

### 同 key 不同 value

策略按 type 选择，并写入 schema：

- **fact**：默认走用户确认 / 显式更正路径。用户 UI / settings 修改、用户原话纠正视为显式确认；非确认场景写为 pending，由产品 UI 发起确认。
- **preference / derived / episodic**：可在以下三种里择一并显式声明：
  - **时间优先**：新值覆盖，旧值 `status=superseded`。
  - **置信度优先**：高 confidence 覆盖，低 confidence 不写。
  - **累积**：多值并存（如 preference 中的"目标语言"可同时学多门）。
- 未列出的策略组合都不允许，避免抽取层 silent 覆盖关键事实。

### 跨 type 冲突

- `fact` vs `derived` 冲突：fact 优先，derived 标 outdated 并降权。
- `preference` vs `episodic` 不应冲突，如出现说明 schema 设计错误。
- `fact` vs `fact` 冲突且双方 confidence 都高：写为 pending，强制走用户确认。

冲突解决必须保留 audit，便于追责和回滚。

## 8. 反例（什么不该做）

- 用户首条消息还没说一句关于自己的话时就提取"用户可能想……"。
- 把当下情绪（`"今天好累"`）写为 fact 或 preference。
- 把用户提问（`"我女儿几岁该上学？"`）当 fact `"用户有女儿"` 直接写入；最多写为 derived 并要求多次出现。
- 抽取时把整段对话原文塞进 evidence（隐私 + 体积爆炸）。
- 默认 confidence 1.0；模型其实没有真实判断能力。
- LLM 抽取 prompt 没有"可拒绝"出口 → 强行编造。
- 抽取时不带白名单 → 关键 key 命名漂移，后续无法稳定 read。
- 抽取触发条件耦合在主 agent loop 里 → 抽取失败影响用户回复。
- 用 long-window batch 把旧的 derived 反复重写覆盖近期更新。
