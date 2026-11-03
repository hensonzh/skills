---
name: agent-memory-design
description: 设计、审查或改造智能体跨会话记忆，确定写入、检索、更新、删除与用户控制规则。
---

# Agent Memory Design

## 设计步骤

1. 明确哪些信息需要跨会话保留；业务事实优先从业务系统读取，可用设置解决的偏好不另建记忆系统。
2. 为每类记忆确定 type、key、user/scope、来源、读写时机、保留期限和删除方式。
3. 定义抽取、校验、存储、检索和注入预算；模型只提出候选，应用按产品规则决定写入。
4. 定义冲突、过期、用户纠正与删除后的缓存/索引处理，验证相关读写路径。

## 判断规则

- `fact` 是已声明或可验证事实；`preference` 是明确偏好；`episodic` 是已发生事件；`derived` 是带证据和置信度的推断。
- 写入前检查类目授权和 scope。健康等高敏信息需相应明确授权；用户提及信息本身不等于授权长期保存。
- 在授权范围内，明确且稳定的事实或偏好通过校验后可写入；推断仅在产品已获授权时存为 derived，不升级成事实。
- 临时状态留在会话或任务记录中。各类记忆按用途设置保留期限，不一律永久保存。
- fact 冲突需显式纠正或确认；用户在设置中修改、主动纠正已算确认。其他类型按声明的版本和冲突策略合并。
- 用户要求忘记时，定位对应信息和 scope，按删除策略处理所有相关类型及其副本；对象不明确时再澄清，不能默认只删最近一条 derived。
- 检索限定当前权限与 scope，只注入必要信息；derived 标为假设，外部易变事实重新读取。
- 提供查看、更正和删除入口；显式“记住/忘记”请求反馈执行结果，后台更新不逐轮播报。删除覆盖存储、索引和缓存，审计按既定保留规则处理。

按任务提供决策、数据流、必要字段和验收场景；评审先列具体问题，不强制完整架构模板。

## 按需参考

- 抽取和冲突处理：[extraction-pipeline.md](references/extraction-pipeline.md)。
- 存储、检索与生命周期：[storage-and-retrieval.md](references/storage-and-retrieval.md)。
- 授权、用户控制与删除：[privacy-and-ux.md](references/privacy-and-ux.md)。
- 质量指标与回归场景：[quality-and-eval.md](references/quality-and-eval.md)。
- 需要完整示例时：[worked-example.md](references/worked-example.md)。
