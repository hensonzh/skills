# 上下文契约示例

按实际链路选取字段，不要求创建下面所有对象；框架已有等价结构时直接复用。状态与恢复规则见 [context-management-reference.md](context-management-reference.md)。

## 当前调用输入

```json
{
  "turn_id": "turn_42",
  "thread_id": "thread_7",
  "received_at": "2026-11-04T08:00:00Z",
  "user_message": "查询订单进度",
  "environment": {"timezone": "Asia/Shanghai"},
  "attachment_refs": []
}
```

身份、租户和权限从认证上下文取得；客户端附带的身份字段仅作待校验输入。当前时间在任务需要时刷新，不用旧环境 hash 替代。

## 运行状态与当前可见内容

```json
{
  "run_id": "run_9",
  "checkpoint_ref": "checkpoint_3",
  "history_ref": "history_17",
  "summary_ref": "summary_2",
  "read_reference_ids": ["policy:v3"],
  "visible_reference_ids": [],
  "pending_action_id": null
}
```

`read_reference_ids` 是曾读取的记录，`visible_reference_ids` 由本次组装内容计算；不能仅因前者包含资料就禁止重读。框架能从消息状态判断可见性时无需维护重复列表。

## 记忆投影

```json
{
  "memory_id": "mem_42",
  "kind": "preference",
  "value": {"language": "zh-CN"},
  "source_kind": "user_explicit"
}
```

记忆服务在读取时校验权限和有效性；本层不规定分类枚举、存储字段或删除方式。需要不确定性、证据或时效说明时再添加，未经认可的数据不能当作系统指令。

## 工具结果与产物

```json
{
  "call_id": "call_5",
  "status": "succeeded",
  "model_payload": {"order_id": "order_8", "status": "shipped"},
  "frontend_payload": {"summary": "订单已发货"},
  "result_ref": "result_5"
}
```

模型与前端 payload 根据各自用途裁剪，不必为了不同消费者复制整个内部返回。产物需要跨轮引用、编辑或恢复时再持久化其 ID、版本和数据；不用回答文本推断执行结果。

副作用请求没有可靠回执时可使用 `unknown` 状态；仅凭网络错误不能填 `failed` 并断言未执行。错误字段按实际恢复需要添加。

## 待确认动作

```json
{
  "action_id": "action_6",
  "version": 1,
  "params": {"order_id": "order_8", "amount": 19.99},
  "authorization_ref": null,
  "idempotency_key": "operation_refund_6",
  "status": "pending",
  "result_ref": null
}
```

此对象用于需要额外授权的动作；已授权操作复用现有执行记录，不为套用示例创建确认页面。取得有效授权后记录 `authorization_ref`，执行前校验其范围。

参数变化时重新校验授权是否覆盖，必要时新建动作或增加版本；幂等键标识同一业务操作，不要求固定拼接 thread/user 字符串。

可区分 pending、authorized、executing、applied、unknown、failed、cancelled、expired；仅采用任务需要的状态。未知结果的查询与重试遵循上下文参考中的副作用恢复规则。
