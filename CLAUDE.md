# CLAUDE.md

本文件是 Claude / AI 助手进入 `pi-mono` 仓库时的必读简报。**权威开发规范见 [`AGENTS.md`](./AGENTS.md)**；本文件仅做要点压缩与架构索引，遇到冲突以 `AGENTS.md` 为准。完整架构详见 [`ARCHITECTURE.md`](./ARCHITECTURE.md)。

---

## 1. 项目是什么

Pi 是一个终端驱动的 AI 编程智能体，同时也是可复用的 LLM/Agent 开发框架。仓库是 npm monorepo，由四个相互独立的包组成：

| 包 | 名称 | 职责 |
|----|------|------|
| `packages/ai` | `@earendil-works/pi-ai` | 统一多供应商 LLM API（OpenAI / Anthropic / Google / Bedrock 等几十家） |
| `packages/agent` | `@earendil-works/pi-agent-core` | 通用 Agent 运行时：循环、事件流、工具执行、上下文压缩、session 持久化 |
| `packages/coding-agent` | `@earendil-works/pi-coding-agent` | `pi` CLI 与 SDK：内置工具、TUI / print / RPC 模式、扩展系统、技能 |
| `packages/tui` | `@earendil-works/pi-tui` | 差分渲染终端 UI 组件库（Editor、Markdown、SelectList、Image …） |

依赖方向（单向、无环）：

```
coding-agent ──> agent ──> ai
            └──> tui
            └──> ai
agent ──> ai
ai / tui 无内部依赖
```

设计原则：包边界清晰、`Context` 可序列化、事件驱动、provider SDK 延迟加载、核心最小 + 扩展强（extensions / skills / prompts / themes）。

---

## 2. 一次交互的数据流（速查）

```
TUI 输入 → coding-agent interactive-mode → AgentSession.prompt()
        → agent-core Agent → agent-loop → pi-ai Models.stream()
        → Provider 路由 + 鉴权 → API implementation → LLM 流式响应
        → AgentEvent 向上传播 → TUI 更新 + 工具执行（read/bash/edit/...）
        → session 写入 JSONL
```

关键入口：
- CLI：`packages/coding-agent/src/cli.ts` → `src/main.ts` → `src/core/agent-session.ts`
- 模型路由：`packages/ai/src/index.ts` → `src/models.ts` → `src/providers/*`
- Agent 循环：`packages/agent/src/agent.ts` → `src/agent-loop.ts`

---

## 3. 红线规则（不可越界）

以下条款摘自 `AGENTS.md`，必须遵守：

### 代码质量
- 大范围改动前**完整读取**目标文件，不要只看搜索片段。
- 不写 `any`（除非绝对必要）；不写编译用的 backward-compatibility 代码（除非 Captain 显式要求）。
- **禁止动态导入**：`await import()` / `import("pkg").Type` / 动态 type import 一律禁止，顶层 import only。
- 受根 tsconfig 约束的代码（`packages/*/src`, `packages/*/test`, `packages/coding-agent/examples`）只能用 Node strip-only 可擦除语法：**禁** parameter properties、`enum`、`namespace`/`module`、`import =`、`export =`。
- 不要硬编码 key 检查（如 `matchesKey(keyData, "ctrl+x")`）；走 `DEFAULT_EDITOR_KEYBINDINGS` / `DEFAULT_APP_KEYBINDINGS`。
- **禁止直接修改** `packages/ai/src/models.generated.ts`；改 `packages/ai/scripts/generate-models.ts` 后重新生成。
- 类型报错时不要降级/删除代码来绕过；升级依赖才是正解。
- 删除/移除看起来有意图的功能前必须先问。

### 命令
- 代码改动后跑 `npm run check`（完整输出，不要 tail），全部错误/警告/info 修干净后才能提交。**不跑测试**。
- 不要主动跑 `npm run build` 或 `npm test`（除非 Captain 要求）。
- 不要直接跑完整 vitest 套件——会触发依赖真实 endpoint/key 的 e2e。非 e2e 测试用仓库根的 `./test.sh`；单测从包根跑 `node ../../node_modules/vitest/dist/cli.js --run test/specific.test.ts`。
- `packages/coding-agent/test/suite/` 一律用 `test/suite/harness.ts` + faux provider，不连真实 provider、不用真实 key。
- issue 回归测试放 `packages/coding-agent/test/suite/regressions/`，命名 `<issue-number>-<short-slug>.test.ts`。
- 临时脚本写到 `/tmp` 跑完即删，不要在 bash 命令里塞多行脚本。

### 依赖与 lockfile
- 外部依赖一律 pinned exact version。
- 本地装依赖用 `npm install --ignore-scripts`；CI 风格用 `npm ci --ignore-scripts`。
- 只刷新 lockfile：`npm install --package-lock-only --ignore-scripts`。
- `packages/coding-agent/npm-shrinkwrap.json` 用 `node scripts/generate-coding-agent-shrinkwrap.mjs` 重生成。新增带 lifecycle script 的依赖必须显式加入 allowlist。
- pre-commit 默认拦截 lockfile 变更，**不要 bypass**，除非 Captain 确认要提交 lockfile。

### Git（多 session 并发，硬性约束）
- **只提交你这次会话改过的文件**。
- 用 `git add <显式路径>`；**禁** `git add -A` / `git add .`。
- 提交前 `git status` 核对。
- `packages/ai/src/models.generated.ts` 可以随你的改动一起带上。
- 提交信息格式：`{feat,fix,docs}[(ai,tui,agent,coding-agent)]: <message>`。
- **禁用**：`git reset --hard`、`git checkout .`、`git clean -fd`、`git stash`、`git add -A`、`git add .`、`git commit --no-verify`、`git push --force`。
- rebase 冲突：只解决你改过的文件；其他文件的冲突直接 abort 问 Captain。
- **未经 Captain 要求，不要 commit**。

### PR / Issue
- 评审 PR 时**不要** `gh pr checkout` / `git switch`；用 `gh pr view` / `gh pr diff` / `gh api` / `git show <ref>:<path>` 读取。
- 创建 issue 加 `pkg:*` 标签（`pkg:ai` / `pkg:agent` / `pkg:coding-agent` / `pkg:tui`）。
- 评论先写临时文件再 `gh issue/pr comment --body-file`，不要把 markdown 直接塞 `--body`。
- 用 commit 关 issue：每个 issue 单独写 `closes #N`（`closes #1, closes #2`），共享关键字只关第一个。

---

## 4. Conversational Style（来自 AGENTS.md）

- 简短、直接，技术散文。
- commit / issue / PR / 代码里不放 emoji。
- 用户提问先答问题，再做改动或跑命令。
- 回应反馈或分析时，先明确说同意或不同意，再说改了什么。

---

## 5. 扩展与配置入口

| 机制 | 位置 |
|------|------|
| Extensions | `~/.pi/agent/extensions/` 或 `.pi/extensions/` |
| Skills | `~/.pi/agent/skills/` 或 `.agents/skills/` |
| Prompt Templates | `~/.pi/agent/prompts/` 或 `.pi/prompts/` |
| Themes | `~/.pi/agent/themes/` 或 `.pi/themes/` |
| 项目级指令 | `AGENTS.md` / `CLAUDE.md`（启动时自动加载） |
| 配置 | `~/.pi/agent/settings.json` / `.pi/settings.json` |
| 自定义模型 | `~/.pi/agent/models.json` |

---

## 6. 常用命令

```bash
npm install --ignore-scripts   # 装依赖（不跑 lifecycle）
npm run check                  # 类型 / 格式 / lint / 依赖校验（代码改动必跑）
./test.sh                      # 非 e2e 测试
./pi-test.sh                   # 从源码跑 pi CLI
```

发布、TUI tmux 测试、CHANGELOG 等流程详见 `AGENTS.md` 对应小节。

---

## 7. User Override

若 Captain 的指令与本文件或 `AGENTS.md` 冲突，**先显式确认**，确认后再执行。
