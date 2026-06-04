---
name: agent-memory-design
description: 设计、评审或改造智能体长期记忆系统时使用，覆盖如何从会话历史提炼信息、记忆类型与质量门槛、存储介质选型（KV/向量/图/混合）、检索与排序、注入 prompt 的格式与预算、生命周期与冲突解决、用户授权与删除 UX、隐私合规、评估指标与测试。
---

# Agent Memory Design

设计 agent 的长期记忆系统：决定从会话里提炼什么、怎么提炼、存到哪里、什么时候读出来、如何维护、如何评估。

适用场景：个人助手、学习 agent、长期客服、Coach、coding agent、健康/习惯类 agent；任何需要跨会话保留与该用户/账号相关稳定知识的产品。

## 核心心智模型

**记忆 ≠ 历史 ≠ 外部事实**：

- **历史**：同一会话内的连续推理上下文，由对话或平台 thread 承接，会话结束自然过期。
- **外部事实**：业务系统、数据库、知识库的事实，按需读取，不归 agent 拥有。
- **记忆**：跨会话保留的、与该用户/账号相关的稳定结构化知识，由 agent 显式管理。

记忆系统由四条管线组成：

1. **写路径**：信号识别 → 抽取 → 校验 → 落库。
2. **读路径**：查询构建 → 检索排序 → 注入预算 → 上下文格式化。
3. **生命周期**：更新、合并、过期、删除、版本。
4. **质量与 UX**：用户授权、可见可改可删、评估与监控。

## 四种记忆类型

| 类型 | 写入门槛 | 典型来源 | 默认保留 | 注入方式 |
| --- | --- | --- | --- | --- |
| **fact** | 高，可验证或用户显式 | 注册资料、explicit 声明、确认动作 | 永久（用户删除前） | 高优先级注入或 read tool |
| **preference** | 中，用户表达意愿 | explicit、settings、反馈 | 永久（可被新值覆盖） | 高优先级注入 |
| **episodic** | 低，事件型 | 会话摘要、动作记录、里程碑 | 滚动摘要 + 时间衰减 | 语义检索 + recency 加权 |
| **derived** | 低，但必须标注来源和置信度 | 模型从对话或行为推断 | TTL + 重确认 | 仅作"假设"注入，不当事实 |

把 derived 当 fact 写入是最常见的 bug。

## 0→1 设计流程

1. 先判断**是否真的需要记忆**。如果偏好可以走 settings、事实可以从业务库读、上下文可以靠会话历史撑住，先别上记忆系统。
2. 列**记忆类型清单**：每条标 type、key、scope、来源、生命周期、注入策略。
3. 选 **extraction 模式**：用户显式 / 同步 in-loop / 异步 post-run / 离线 batch（详见 [extraction-pipeline.md](references/extraction-pipeline.md)）。
4. 选 **storage**：KV、向量、图、混合，按类型分别落地（详见 [storage-and-retrieval.md](references/storage-and-retrieval.md)）。
5. 设计 **read path**：什么场景查、查什么类型、怎么排序、注多少 token、注入格式。
6. 设计 **lifecycle**：合并、覆盖、TTL、用户删除、冲突解决。
7. 设计 **UX 与隐私**：授权方式、可见、可改、可删（详见 [privacy-and-ux.md](references/privacy-and-ux.md)）。
8. 设计 **eval**：写读质量指标、回归集、监控告警（详见 [quality-and-eval.md](references/quality-and-eval.md)）。

不确定时先看 [worked-example.md](references/worked-example.md) 的端到端样例。

## 决策速查

| 场景 | 默认 |
| --- | --- |
| 用户说"记住我喜欢 X" | preference，立即写，回写 ack 给用户 |
| 用户说"我对 X 过敏" | fact，立即写，需高置信，建议二次确认 |
| 模型推断"用户可能在备孕" | derived，需明确产品授权；否则不写 |
| 上一次会话讨论了什么 | episodic，运行结束后异步摘要 |
| 用户在 settings 改名字 | fact，覆盖旧值，version+1，保留 audit |
| 用户说"忘掉刚才那句" | 软删除最近一条相关 derived；fact/preference 走显式删除流程 |
| 跨设备登录 | 同 user_id 共享；冲突按 updated_at 取新 + version 单调 |
| 高敏数据（医疗、财务、未成年人） | 默认不写，需显式产品授权和合规审查 |
| 一次性任务细节 / 临时情绪 | 永远不写 |

## 必守边界

- 不在没有产品授权时静默写入 derived。
- 不把 episodic 全文当 fact 注入；进 prompt 前必须摘要。
- 不在用户没确认前覆盖 fact；旧值进 audit。用户在 UI / settings 的显式修改、用户原话主动纠正都视为确认；时间优先 / 置信度优先策略只用于 preference / derived / episodic。
- 不让模型自己决定写入条件；写入条件由抽取层与产品规则控制。
- 不读取超出当前 user/scope 的记忆。
- 不把记忆原文（含 evidence）直接展示给前端，除非用户在管理界面查看。
- 不仅做软删除；合规场景必须真物理删除并按规则保留审计。
- 不让记忆替代 RAG / read tool；事实型大数据应留在业务系统。
- 不在每轮回复里报告"我刚刚记住了什么"；通知放管理面板。

## 输出格式

设计或评审一份记忆方案时，建议输出：

1. **是否需要记忆**：列出替代方案（settings、business DB、session-only）后的判断。
2. **记忆清单**：每条 type、key、scope、来源、读写时机、生命周期。
3. **架构图**：写路径、读路径、storage、lifecycle 任务。
4. **Schema**：memory record、extraction job、retrieval query、injection block。
5. **隐私与 UX**：授权、可见性、编辑/删除、合规接触点。
6. **质量与评估**：抽取/注入指标、回归集、监控告警。

## Reference 分工

- [extraction-pipeline.md](references/extraction-pipeline.md)：信号识别、触发模式、抽取方法、抽取 prompt 模板、校验、去重与冲突、反例。
- [storage-and-retrieval.md](references/storage-and-retrieval.md)：介质选型、scope/索引、TTL、检索路径、排序公式、注入预算与格式、读时脱敏、read-side schema。
- [quality-and-eval.md](references/quality-and-eval.md)：写读质量指标、评估方法、运行时监控、回归测试套件、反指标。
- [privacy-and-ux.md](references/privacy-and-ux.md)：授权模型、默认值、管理 UI、编辑/删除、合规接触点、跨设备、共享场景、安全边界。
- [worked-example.md](references/worked-example.md)：个人学习助手 LinguaCoach 0→1 端到端样例（记忆清单、写读路径、schema、12 条验收测试、监控、演进）。
