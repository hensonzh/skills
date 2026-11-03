---
name: cozie-android-device-delivery
description: Cozie App 修改检查通过后，按已确认偏好构建真实后端 Android 包并安装到连接的 Xiaomi 真机。
---

# Cozie Android 真机交付

- 适用于 `deepcozy/lab_mobile` 及关联工作树。App 修改完成、检查通过并本地提交后构建安装；这是持续授权，无需重复询问。
- 纯文档、指令和 Skill 修改不构建 App。用户当次要求优先；本偏好不授权推送或发布。
- 每次发现当前设备，选择在线、已授权的 Xiaomi，排除模拟器；型号 `25098PN5AC` 仅作线索，多个候选无法区分时询问。
- 使用真实后端：`EXPO_PUBLIC_ENABLE_MOCK_ACCOUNT=false`、`EXPO_PUBLIC_START_AT_LOGIN=false`，保留数据和现有会话。
- 不自动启动或更新模拟器；手机未连接、未授权或拒绝安装时报告阻塞并保留已有产物，不换设备、不循环重试提示。
- 签名冲突先报告，不通过卸载或清数据绕过。仅在用户要求或任务验证需要时启动 App。
- SDK、JDK、ABI、预构建、bundle 刷新及安装验证，按当前工作树 `.agents/skills/cozie-app-workflow/references/android.md` 执行；不将个人设备偏好复制到项目通用流程。
- 分别汇报提交、构建、目标手机、安装结果、页面验证和未验证项。
