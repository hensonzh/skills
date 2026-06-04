# Agent Loop Streaming Skill

这是一个用于设计和实现 **智能体 loop 流式输出体验** 的 skill。它帮助产品经理、独立开发者、vibe coder 和 coding agent，把“用户应该看到智能体正在工作”这类模糊需求，转成可以落地的前端体验、事件契约、UI 状态、结构化 artifact 渲染规则和验收标准。

适用场景包括：

- 你不想让用户只看到一个 spinner，希望展示 agent 正在理解、检索、调用工具、生成结果或等待确认。
- 你的 agent 会调用工具、查资料、生成报告卡片、表单、计划、草稿或工单。
- 你的前端现在把工具 JSON、中间文本和最终回答混在一起，用户看不懂 agent 到底在做什么。
- 你想把产品想法整理成 Codex、Claude Code、Cursor 等代码助手可以直接实现的需求。

## 目录结构

```text
agent-loop-streaming/
├── SKILL.md
├── README.md
├── evals/
│   └── evals.json
└── references/
    ├── debugging-and-tests.md
    ├── event-contract.md
    └── frontend-work-panel.md
```

各文件作用：

- `SKILL.md`：skill 的触发说明、核心心智模型、使用方式、输出格式和体验原则。
- `references/frontend-work-panel.md`：前端体验模式，包括 chat transcript、work panel、artifact area、action area 的信息层级。
- `references/event-contract.md`：应用侧事件模型、事件到 UI 的映射、按 `tool_call_id` 合并 work step 的建议。
- `references/debugging-and-tests.md`：给 vibe coder 和 coding agent 的实现 brief 模板、调试建议和验收清单。
- `evals/evals.json`：用于检查这个 skill 是否按预期工作的示例评测 prompt。

## 快速安装思路

最适合非专业程序员的方式不是自己手动移动文件，而是：

1. 先把这个仓库下载到电脑里。
2. 打开你正在使用的 Codex、Claude Code 或 Cursor。
3. 把下面对应工具的安装 prompt 发给它。
4. 让它帮你检查目录、移动文件、确认是否安装成功。

如果你电脑已经安装 git，可以 clone 仓库：

```bash
git clone https://github.com/hensonzh/agent-loop-streaming.git
```

如果你电脑没有安装 git，可以直接打开 GitHub 仓库页面，点击右上方绿色的 `Code` 按钮，然后点击 `Download ZIP` 下载。下载后把 ZIP 解压，得到 `agent-loop-streaming` 文件夹。

如果你不知道 clone 或解压到了哪里，可以直接把这句话发给你的 AI 助手：

```text
我刚刚下载了 https://github.com/hensonzh/agent-loop-streaming.git，但我不知道它在哪个目录。请帮我在常见下载目录和当前项目附近查找 `agent-loop-streaming` 文件夹，然后按你的工具要求帮我安装。
```

## 添加到 Codex

### 推荐方式（面向非专业程序员）：让 Codex 帮你安装

如果你已经在 Codex 里，可以把下面这段话直接发给 Codex：

```text
请帮我安装 agent-loop-streaming skill。

我已经下载或 clone 了这个仓库：
https://github.com/hensonzh/agent-loop-streaming.git

请你做这些事：
1. 先判断我的系统是 macOS、Windows 还是 Linux。
2. 在当前项目、下载目录、桌面、用户目录等常见位置查找 `agent-loop-streaming` 文件夹。
3. 如果找到的是 `agent-loop-streaming-main`，请把它当作下载后的 skill 文件夹处理。
4. 确认里面有 `SKILL.md`、`references/` 和 `evals/`。
5. 按当前系统创建 Codex skills 目录：macOS/Linux 用 `~/.codex/skills/`，Windows 用 `%USERPROFILE%\.codex\skills\`。
6. 把整个 skill 文件夹复制进去，并命名为 `agent-loop-streaming`。
7. 如果目标目录已经存在，先告诉我，再询问是否覆盖。
8. 安装后确认 `agent-loop-streaming/SKILL.md` 存在。
```

安装完成后，重启 Codex，或开一个新的 Codex 会话。

### 手动方式（面向专业程序员）

macOS / Linux：

```bash
mkdir -p ~/.codex/skills
cp -R ~/Downloads/agent-loop-streaming ~/.codex/skills/agent-loop-streaming
```

Windows PowerShell：

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.codex\skills"
Copy-Item -Recurse "$env:USERPROFILE\Downloads\agent-loop-streaming" "$env:USERPROFILE\.codex\skills\agent-loop-streaming"
```

如果你是通过 `Download ZIP` 下载的，解压后的文件夹可能叫 `agent-loop-streaming-main`。这种情况下，把命令里的 `agent-loop-streaming` 换成 `agent-loop-streaming-main`。

## 添加到 Cursor

Cursor 目前更常用的是 **Rules** 机制，而不是直接读取 `SKILL.md` 作为 Claude/Codex 风格的 skill。因此建议把这个 skill 转成 Cursor 项目规则。

### 推荐方式（面向非专业程序员）：让 Cursor 帮你安装

如果你已经在 Cursor 里打开了自己的项目，可以把下面这段话发给 Cursor Chat：

```text
请帮我把 agent-loop-streaming skill 安装成当前项目的 Cursor Rule。

我已经下载或 clone 了这个仓库：
https://github.com/hensonzh/agent-loop-streaming.git

请你做这些事：
1. 先判断我的系统是 macOS、Windows 还是 Linux。
2. 在当前项目、下载目录、桌面、用户目录等常见位置查找 `agent-loop-streaming` 文件夹。
3. 如果找到的是 `agent-loop-streaming-main`，请把它当作下载后的 skill 文件夹处理。
4. 确认里面有 `SKILL.md`、`references/` 和 `evals/`。
5. 在当前项目创建 `.cursor/rules/` 目录。
6. 基于 `agent-loop-streaming/SKILL.md` 创建 `.cursor/rules/agent-loop-streaming.mdc`。
7. 在这个 `.mdc` 文件顶部加入 Cursor rule frontmatter：
   `description: 设计、实现或调试 agent loop streaming 前端体验时使用，包括 work panel、应用侧事件、artifact 渲染、确认动作和验收标准。`
   `alwaysApply: false`
8. 把整个 `agent-loop-streaming` 文件夹复制到当前项目的 `docs/skills/agent-loop-streaming/`，让 references 也能被 Cursor 读取。
9. 在 `.cursor/rules/agent-loop-streaming.mdc` 里补一句：需要更多细节时，读取 `docs/skills/agent-loop-streaming/references/` 下的参考文档。
10. 安装后确认 `.cursor/rules/agent-loop-streaming.mdc` 和 `docs/skills/agent-loop-streaming/SKILL.md` 存在。
```

### 手动方式（面向专业程序员）

macOS / Linux：

```bash
mkdir -p .cursor/rules docs/skills
cp ~/Downloads/agent-loop-streaming/SKILL.md .cursor/rules/agent-loop-streaming.mdc
cp -R ~/Downloads/agent-loop-streaming docs/skills/agent-loop-streaming
```

Windows PowerShell：

```powershell
New-Item -ItemType Directory -Force ".cursor\rules"
New-Item -ItemType Directory -Force "docs\skills"
Copy-Item "$env:USERPROFILE\Downloads\agent-loop-streaming\SKILL.md" ".cursor\rules\agent-loop-streaming.mdc"
Copy-Item -Recurse "$env:USERPROFILE\Downloads\agent-loop-streaming" "docs\skills\agent-loop-streaming"
```

然后打开 `.cursor/rules/agent-loop-streaming.mdc`，在文件顶部补上 Cursor rule 的说明：

```md
---
description: 设计、实现或调试 agent loop streaming 前端体验时使用，包括 work panel、应用侧事件、artifact 渲染、确认动作和验收标准。
alwaysApply: false
---
```

如果你是通过 `Download ZIP` 下载的，解压后的文件夹可能叫 `agent-loop-streaming-main`。这种情况下，把命令里的 `agent-loop-streaming` 换成 `agent-loop-streaming-main`。

## 添加到 Claude Code

### 推荐方式（面向非专业程序员）：让 Claude Code 帮你安装

如果你已经在 Claude Code 里，可以把下面这段话直接发给 Claude Code：

```text
请帮我安装 agent-loop-streaming skill。

我已经下载或 clone 了这个仓库：
https://github.com/hensonzh/agent-loop-streaming.git

请你做这些事：
1. 先判断我的系统是 macOS、Windows 还是 Linux。
2. 在当前项目、下载目录、桌面、用户目录等常见位置查找 `agent-loop-streaming` 文件夹。
3. 如果找到的是 `agent-loop-streaming-main`，请把它当作下载后的 skill 文件夹处理。
4. 确认里面有 `SKILL.md`、`references/` 和 `evals/`。
5. 按当前系统创建 Claude Code skills 目录：macOS/Linux 用 `~/.claude/skills/`，Windows 用 `%USERPROFILE%\.claude\skills\`。
6. 把整个 skill 文件夹复制进去，并命名为 `agent-loop-streaming`。
7. 如果目标目录已经存在，先告诉我，再询问是否覆盖。
8. 安装后确认 `agent-loop-streaming/SKILL.md` 存在。
```

安装完成后，重启 Claude Code，或退出当前会话后重新进入项目。

### 手动方式（面向专业程序员）

macOS / Linux：

```bash
mkdir -p ~/.claude/skills
cp -R ~/Downloads/agent-loop-streaming ~/.claude/skills/agent-loop-streaming
```

Windows PowerShell：

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.claude\skills"
Copy-Item -Recurse "$env:USERPROFILE\Downloads\agent-loop-streaming" "$env:USERPROFILE\.claude\skills\agent-loop-streaming"
```

如果你是通过 `Download ZIP` 下载的，解压后的文件夹可能叫 `agent-loop-streaming-main`。这种情况下，把命令里的 `agent-loop-streaming` 换成 `agent-loop-streaming-main`。

### 项目级安装

如果你只想让某一个项目使用这个 skill，可以让 Claude Code 放在项目目录下。把下面这段话发给 Claude Code：

```text
请把 agent-loop-streaming 安装成当前项目专用的 Claude Code skill。

请你找到我下载或 clone 的 `agent-loop-streaming` 文件夹，然后复制到当前项目的 `.claude/skills/agent-loop-streaming/`。
如果 `.claude/skills/agent-loop-streaming/` 已经存在，请先问我是否覆盖。
安装后请确认 `.claude/skills/agent-loop-streaming/SKILL.md` 存在。
```

## 安全建议

安装任何第三方 skill、rule 或提示词包之前，都建议先打开 `SKILL.md` 看一遍。它会影响 coding agent 如何理解任务、读取文件、提出修改建议，甚至可能影响它是否建议执行某些命令。

这个 skill 不包含可执行脚本，主要是 Markdown 指令和参考文档；但你仍然应该在安装前确认内容符合你的团队习惯和安全边界。

## 适用范围

这个 skill 不绑定任何具体前端框架，也不要求你使用某一家模型服务商。它关注的是更稳定的产品和实现契约：

- 用户应该看到哪些阶段。
- 中间文本和最终回答如何分离。
- 工具调用如何映射成用户能理解的 work step。
- artifact 应该在哪里渲染。
- 有副作用的动作如何 preview 和 confirmation。
- 前端如何通过应用侧事件更新状态，而不是直接展示 provider 原始 event。

它可以用于 React、Vue、Svelte、Next.js、原生 Web、移动端 WebView，以及任何需要展示 agent loop 过程的产品界面。
