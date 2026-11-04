---
name: agent-memory-design
description: 设计、审查或改造智能体跨会话记忆，确定写入、检索、更新、删除与用户控制规则。
---

# Agent Memory Design

## 执行规则

- 先确定跨会话保留的用途；业务事实从业务系统读取，可用设置解决的偏好不另建记忆系统，临时状态留在任务记录中。
- 按用途选择分类、存储和检索方式；明确身份与权限作用域、来源、更新与删除规则，不固定四类记忆或多数据库架构。
- 写入前检查授权与来源；高敏信息需相应明确授权，用户提及不等于允许长期保存。模型只提出候选，应用校验后写入。
- 区分明确事实、偏好、事件和推断；推断须在授权范围内保留证据与不确定性，不升级为事实或系统指令。
- 事实冲突需显式纠正或确认；用户主动纠正或修改设置已算确认。其他更新按声明的版本与冲突策略处理。
- 检索限定权限、相关性和预算；记忆作为带来源的数据注入，易变事实重新读取，过期或删除内容退出有效检索。
- 提供查看、更正和删除入口；“忘记”覆盖对应信息及相关副本，删除与保留策略按产品约定，避免后台任务重新写回。
- 显式记住、纠正或忘记请求反馈结果，后台更新不逐轮播报；按受影响路径验证读写质量，不强制固定阈值或 A/B。

只输出本次需要的决策、字段与验证；完整示例按需读取。

## 按需参考

- 抽取和冲突处理：[extraction-pipeline.md](references/extraction-pipeline.md)。
- 存储、检索与生命周期：[storage-and-retrieval.md](references/storage-and-retrieval.md)。
- 授权、用户控制与删除：[privacy-and-ux.md](references/privacy-and-ux.md)。
- 质量指标与回归场景：[quality-and-eval.md](references/quality-and-eval.md)。
- 需要完整示例时：[worked-example.md](references/worked-example.md)。
