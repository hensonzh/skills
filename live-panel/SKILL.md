---
name: live-panel
description: 将系统、流程或智能体架构制作成持续运行的动画面板，输出 MP4 或循环网页；不用于真实监控或交互式图表。
---

# Live Panel

以下路径相对本 Skill 目录；产物写入任务输出目录。布局保持固定，用数据包、日志和状态变化表达运行过程。

## 制作面板

1. 明确节点、连线、状态、输出尺寸和数据来源；无真实来源的数字标为 illustrative。
2. 选择现有配置：循环用 ring，逐级转化用 funnel，带判断和回路的流程用 loop，手绘风格用 excalidraw；其他布局用自由元素。nebula、timeline、tree、kpi 尚未实现。
3. 行列布局用 `python3 scripts/scaffold.py --spec spec.json --out out/config.json` 生成初稿；`--example` 查看输入格式。特殊布局复制最接近的示例后修改，避免一次手写大段 JSON。
4. 按需读取 [配置字段](references/config-schema.md) 和 [动画规则](references/motion-grammar.md)。各处数值来自同一状态，`seek(t)` 可重复重放，模拟数据与来源写入顶层 `credit`。
5. 自由布局运行 `python3 scripts/check_frames.py --config out/config.json --out-dir out/frames --repeat`；使用现有 view 时运行 `python3 scripts/verify_view.py --config out/config.json --combos cur --out out/check`。修正报错并查看生成的截图。
6. MP4 用 `python3 scripts/render.py --config out/config.json --out out/panel.mp4`；同时保留网页时加 `--html-out out/panel.html`。需 Chrome/Chromium 与 ffmpeg，缺失时报告具体依赖。
7. 交付产物及实际检查结果；尺寸或主题变化后验证对应配置，不将几何检查当作视觉验收。

## 按需处理

- 导入 `.excalidraw`：先运行 `python3 scripts/excalidraw_import.py input.excalidraw -o out/config.json`。缺少中文手写字体时用 `scripts/fetch_fonts.sh`。
- 新增或修改通用 view：读取 [扩展契约](references/views.md)、[视图规则](references/view-rules.md) 和 [验证说明](references/verify-view.md)。运行全部 7 个组合：当前配置加 3 种画布 × 2 种主题。
- 示例位于 `examples/`；模板与运行库位于 `assets/`，制作普通面板时修改配置即可。
- 复刻他人图片时在 `credit` 和发布文案保留作者及用户提供的完整来源链接。Codex 示例的作者与许可说明见 [NOTICE](NOTICE)；保留相关署名。
