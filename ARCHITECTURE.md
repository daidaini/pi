# Pi 项目架构总览

本文档面向刚接触 `pi-mono` 仓库的开发者，用一张图和分层说明帮你快速理解项目结构、核心模块与数据流。

---

## 1. 项目定位

Pi 是一个**终端驱动的 AI 编程智能体（coding agent）**，同时也是一个可复用的智能体/LLM 开发框架。它由四个相互独立的 npm 包组成一个 monorepo，核心设计目标是：

- **最小核心 + 强扩展**：核心只保留读写文件、bash、编辑等基础工具；子代理、计划模式、权限门等通过扩展实现。
- **统一多供应商 LLM API**：同一套接口调用 OpenAI、Anthropic、Google、Amazon Bedrock 等几十家模型。
- **可嵌入**：既可作为 `pi` CLI 使用，也可通过 SDK 嵌入到其他 Node.js 应用中。

---

## 2. 高层架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              用户 / 其他进程                                  │
│   终端输入 ──────┬────── stdin/stdout JSONL ──────┬────── npm import        │
│                 │                                  │                        │
│                 ▼                                  ▼                        │
│   ┌─────────────────────────┐   ┌─────────────────────────┐                 │
│   │   交互式 TUI / CLI       │   │   RPC / JSON / SDK      │                 │
│   │   (packages/coding-agent)│   │   (packages/coding-agent)│                │
│   └───────────┬─────────────┘   └───────────┬─────────────┘                 │
└───────────────┼───────────────────────────────┼─────────────────────────────┘
                │                               │
                ▼                               ▼
        ┌─────────────────────────────────────────────────────┐
        │           Agent 运行时层                             │
        │   @earendil-works/pi-agent-core                      │
        │   (packages/agent)                                   │
        │   • Agent 循环 / 事件流 / 工具执行 / 上下文压缩          │
        │   • Session 持久化（JSONL）/ 分支 / 技能系统            │
        └─────────────────────────┬───────────────────────────┘
                                  │
                                  ▼
        ┌─────────────────────────────────────────────────────┐
        │           统一 LLM API 层                            │
        │   @earendil-works/pi-ai                              │
        │   (packages/ai)                                      │
        │   • Providers / Models / API implementations          │
        │   • 鉴权、OAuth、工具schema、图片生成                  │
        └─────────────────────────┬───────────────────────────┘
                                  │
                ┌─────────────────┼─────────────────┐
                ▼                 ▼                 ▼
        ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
        │  OpenAI     │   │  Anthropic  │   │   Google    │
        │  xAI        │   │  Mistral    │   │   Vertex    │
        │  Groq       │   │  DeepSeek   │   │   Bedrock   │
        │  ...        │   │  ...        │   │   ...       │
        └─────────────┘   └─────────────┘   └─────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                              底层支撑层                                       │
│   @earendil-works/pi-tui (packages/tui)                                      │
│   • 差分渲染终端 UI 组件库（Editor、Markdown、SelectList、Image 等）            │
│   • 键盘输入、光标/IME、Kitty/iTerm2 内联图片                                 │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 包依赖关系

```
packages/coding-agent
        │
        ├── depends on ──> packages/agent
        │                    │
        ├── depends on ──> packages/ai  <────── (agent 也依赖 ai)
        │
        └── depends on ──> packages/tui

packages/agent
        │
        └── depends on ──> packages/ai

packages/ai          (无内部依赖，纯 LLM 适配层)
packages/tui         (无内部依赖，纯终端 UI 组件库)
```

---

## 4. 各包职责详解

### 4.1 `packages/ai` — `@earendil-works/pi-ai`

**定位**：统一多供应商 LLM API。

| 目录/模块 | 说明 |
|----------|------|
| `src/index.ts` | 核心导出：`Models`、`createModels`、类型、工具验证等 |
| `src/providers/` | 每个供应商一个 provider（OpenAI、Anthropic、Google、Bedrock、OpenRouter 等），包含模型目录与鉴权配置 |
| `src/providers/all.ts` | `builtinModels()` / `builtinImagesModels()` 注册全部内置供应商 |
| `src/api/` | 底层协议实现：`anthropic-messages`、`openai-responses`、`openai-completions`、`google-generative-ai`、`bedrock-converse-stream` 等 |
| `src/api/*.lazy.ts` | 延迟加载 SDK 的包装，首次请求时才 import 供应商 SDK |
| `src/auth/` | 鉴权上下文、凭据存储、环境变量解析 |
| `src/utils/oauth/` | OAuth 登录流程（Anthropic、GitHub Copilot、OpenAI Codex、Vertex） |
| `src/images*.ts` | 图片生成相关接口与模型目录 |
| `src/models.generated.ts` | 脚本生成的模型元数据 |
| `scripts/generate-models.ts` | 模型元数据生成器，不要手动改 `.generated.ts` |

**关键抽象**：

- `Models`：provider 集合，提供 `stream(model, context)`、`complete(model, context)`、`getModel(provider, id)`。
- `Context`：可序列化的对话上下文 `{ systemPrompt, messages, tools }`。
- `Provider`：包含自己的模型列表、鉴权、API 实现；provider 之间可以无缝切换（cross-provider handoff）。
- `Tool`：基于 `typebox` schema 定义，自动校验。

### 4.2 `packages/agent` — `@earendil-works/pi-agent-core`

**定位**：通用 Agent 运行时，管理 Agent 循环、事件流、状态、工具执行。

| 目录/模块 | 说明 |
|----------|------|
| `src/agent.ts` | `Agent` 类，封装高层 prompt/continue 接口 |
| `src/agent-loop.ts` | 底层 `agentLoop` / `agentLoopContinue`，生成事件流 |
| `src/types.ts` | 核心类型：`AgentTool`、`AgentMessage`、`AgentEvent` 等 |
| `src/harness/` | Agent 的“马具”层：session、compaction、skills、prompt templates、system prompt |
| `src/harness/session/` | Session 存储实现：JSONL 仓库、内存仓库、UUID 生成 |
| `src/harness/compaction/` | 上下文压缩：摘要生成、分支摘要、token 估算 |
| `src/harness/skills.ts` | [Agent Skills](https://agentskills.io) 规范实现 |
| `src/proxy.ts` | 浏览器/后端代理工具 |

**关键抽象**：

- `Agent`：接收 prompt，产出事件流；支持 steering、follow-up、abort。
- `AgentTool`：`execute(toolCallId, params, signal, onUpdate)`，返回 content/details，可设置 `terminate: true`。
- `AgentMessage`：可扩展消息类型，通过 declaration merging 支持自定义角色。
- 事件流：`agent_start` → `turn_start` → `message_start`/`message_update`/`message_end` → `tool_execution_*` → `turn_end` → `agent_end`。

### 4.3 `packages/coding-agent` — `@earendil-works/pi-coding-agent`

**定位**：面向终端用户的 `pi` CLI 与 SDK，把 `agent-core` 包装成可交互的编码助手。

| 目录/模块 | 说明 |
|----------|------|
| `src/cli.ts` / `src/main.ts` | CLI 入口 |
| `src/cli/args.ts` | 命令行参数解析 |
| `src/cli/startup-ui.ts` | 启动 UI、信任提示 |
| `src/core/agent-session.ts` | 面向用户的 `AgentSession`：封装 prompt、事件、session 管理 |
| `src/core/agent-session-runtime.ts` | 可替换的运行时（多 session、自定义依赖） |
| `src/core/agent-session-services.ts` | 服务组装（model registry、auth storage、session manager） |
| `src/core/tools/` | 内置工具：`read`、`write`、`edit`、`edit-diff`、`bash`、`grep`、`find`、`ls` |
| `src/core/extensions/` | 扩展系统：加载、运行、类型 |
| `src/core/skills.ts` | 技能加载与解析 |
| `src/core/session-manager.ts` | session 文件管理 |
| `src/core/settings-manager.ts` | 全局/项目级 `settings.json` |
| `src/core/compaction/` | coding-agent 侧的上下文压缩 |
| `src/core/export-html/` | `/export` 导出 HTML |
| `src/modes/interactive/` | 交互式 TUI 模式 |
| `src/modes/print-mode.ts` | `-p` / `--print` 非交互模式 |
| `src/modes/rpc/` | `--mode rpc` JSONL over stdio |
| `src/utils/` | 通用工具：ANSI、剪贴板、图片处理、Git、shell、语法高亮等 |
| `src/bun/` | Bun 单二进制构建入口 |
| `docs/` | 用户文档 |
| `examples/` | 扩展示例：Doom、子代理、计划模式、Gondolin 沙箱等 |

**关键抽象**：

- `AgentSession`：一个可交互的编码 session，用户通过它发送 prompt、订阅事件。
- `ModelRegistry`：维护可用模型列表，结合 `AuthStorage` 管理鉴权。
- `ExtensionAPI`：扩展可注册工具、命令、快捷键、事件监听器、UI 组件。
- `Skill`：Markdown 文件，按 Agent Skills 规范描述能力，通过 `/skill:name` 触发。

### 4.4 `packages/tui` — `@earendil-works/pi-tui`

**定位**：差分渲染的终端 UI 组件库，独立可复用。

| 目录/模块 | 说明 |
|----------|------|
| `src/tui.ts` | `TUI` 容器：组件树、焦点、叠加层、渲染调度 |
| `src/terminal.ts` | `Terminal` 接口及 `ProcessTerminal`、`VirtualTerminal` 实现 |
| `src/components/` | 组件：`Editor`、`Markdown`、`SelectList`、`Input`、`Image`、`Loader`、`Box` 等 |
| `src/keys.ts` / `src/keybindings.ts` | 键盘输入解析、快捷键绑定 |
| `src/autocomplete.ts` | 编辑器自动补全（文件路径、斜杠命令） |
| `src/terminal-image.ts` | Kitty / iTerm2 内联图片协议 |

---

## 5. 一次交互的数据流

以用户在交互式 TUI 中输入“帮我重构 src/index.ts”为例：

```
1. TUI 捕获键盘输入 → Editor 组件提交文本
   └─ packages/tui/src/components/editor.ts

2. coding-agent 的 interactive-mode 把文本包装成 AgentMessage
   └─ packages/coding-agent/src/modes/interactive/interactive-mode.ts

3. AgentSession.prompt() 调用 agent-core 的 Agent 类
   └─ packages/coding-agent/src/core/agent-session.ts
   └─ packages/agent/src/agent.ts

4. Agent 进入 agent-loop，调用 pi-ai 的 Models.stream()
   └─ packages/agent/src/agent-loop.ts
   └─ packages/ai/src/models.ts

5. Models 根据 model.provider 路由到对应 Provider，解析鉴权，调用 API implementation
   └─ packages/ai/src/providers/*.ts
   └─ packages/ai/src/api/*.ts

6. 收到 LLM 的流式响应：文本、思考块、工具调用参数片段
   └─ packages/ai/src/utils/event-stream.ts

7. agent-loop 把事件转成 AgentEvent，向上传播
   └─ packages/agent/src/agent-loop.ts

8. coding-agent 的 interactive-mode 订阅事件，更新 TUI 消息列表
   └─ packages/coding-agent/src/modes/interactive/interactive-mode.ts

9. 若 LLM 调用 read/bash/edit 等工具：
   a. Agent 触发 tool_execution_start
   b. coding-agent 的 tools/*.ts 执行真实操作
   c. 结果作为 toolResult 消息回到上下文
   d. Agent 自动继续下一轮 LLM 调用

10. 最终 assistant 消息完成，session 保存到 JSONL
    └─ packages/agent/src/harness/session/jsonl-repo.ts
```

---

## 6. 关键目录与入口速查

| 用途 | 路径 |
|------|------|
| 根 package.json / 脚本 | `package.json` |
| 开发规范 | `AGENTS.md` |
| 贡献指南 | `CONTRIBUTING.md` |
| AI 包入口 | `packages/ai/src/index.ts` |
| Agent 包入口 | `packages/agent/src/index.ts` |
| coding-agent 包入口 | `packages/coding-agent/src/index.ts` |
| CLI 入口 | `packages/coding-agent/src/cli.ts` |
| TUI 包入口 | `packages/tui/src/index.ts` |
| 模型元数据生成脚本 | `packages/ai/scripts/generate-models.ts` |
| 发布脚本 | `scripts/release.mjs` |
| 本地发布测试 | `scripts/local-release.mjs` |
| CI 工作流 | `.github/workflows/` |

---

## 7. 扩展与自定义入口

Pi 把大量能力外置，核心只提供运行框架。新功能通常通过以下机制加入：

| 机制 | 位置 | 说明 |
|------|------|------|
| **Extensions** | `~/.pi/agent/extensions/` 或 `.pi/extensions/` | TypeScript 模块，可注册工具、命令、UI、事件钩子 |
| **Skills** | `~/.pi/agent/skills/` 或 `.agents/skills/` | Markdown 文件，按 Agent Skills 规范描述 |
| **Prompt Templates** | `~/.pi/agent/prompts/` 或 `.pi/prompts/` | `/name` 触发的可复用提示 |
| **Themes** | `~/.pi/agent/themes/` 或 `.pi/themes/` | 主题 JSON |
| **Pi Packages** | npm/git | 把 extensions/skills/prompts/themes 打包分享 |
| **AGENTS.md / CLAUDE.md** | 项目目录或父目录 | 项目级指令，启动时自动加载 |
| **settings.json** | `~/.pi/agent/settings.json` / `.pi/settings.json` | 用户/项目配置 |
| **models.json** | `~/.pi/agent/models.json` | 自定义模型与 provider |

---

## 8. 开发常用命令

```bash
# 安装依赖（不运行 lifecycle scripts）
npm install --ignore-scripts

# 构建全部包（顺序：tui → ai → agent → coding-agent）
npm run build

# 类型检查、格式化、lint、依赖校验
npm run check

# 运行非 e2e 测试
./test.sh

# 从源码运行 pi CLI
./pi-test.sh
```

---

## 9. 阅读顺序建议

刚接触项目时建议按以下顺序阅读：

1. 本文档 `ARCHITECTURE.md`（你现在读的）
2. 根 `README.md` 了解项目定位
3. `AGENTS.md` 了解开发规范
4. `packages/ai/README.md` 了解统一 LLM API 用法
5. `packages/agent/README.md` 了解 Agent 循环与事件
6. `packages/coding-agent/README.md` 了解 CLI 功能与扩展系统
7. `packages/tui/README.md` 了解终端 UI 组件
8. 从 `packages/coding-agent/src/cli.ts` → `src/main.ts` → `src/core/agent-session.ts` 跟一次 CLI 启动流程
9. 从 `packages/ai/src/index.ts` → `src/models.ts` → 一个 provider（如 `src/providers/anthropic.ts`）了解模型路由
10. 从 `packages/agent/src/agent.ts` → `src/agent-loop.ts` 了解 Agent 循环

---

## 10. 核心设计原则

- **包边界清晰**：`ai` 只做 LLM 调用；`agent` 只做 Agent 循环；`coding-agent` 只做终端产品化；`tui` 只做终端 UI。
- **Context 可序列化**：`Context` 对象可 JSON 化，支持跨 provider 切换、session 持久化、RPC。
- **事件驱动**：从 LLM 流到工具执行再到 UI 更新，全部基于事件流，便于扩展和测试。
- **延迟加载**：provider SDK 在首次请求时才加载，减少启动开销。
- **安全默认最小化**：不内置权限系统，鼓励容器化/沙箱化；扩展可任意执行代码。
