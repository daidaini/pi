# AgentHarness 生命周期

`AgentHarness` 是位于低层级 Agent 循环之上的编排层。它负责会话持久化、运行时配置、资源解析、操作锁定以及面向扩展的变更语义。

本文档描述了当前的发展方向和已实现的行为。一些关于扩展/会话外观（session-facade）的细节已在计划中并有明确标注。

## 最终生命周期目标

Harness 的监听器和钩子应当能够闭包引用 `AgentHarness` 实例，并在文档允许的任何事件中调用公共 Harness API。这些调用绝不能破坏正在进行的轮次快照、重排已持久化的对话记录条目、丢失待处理写入、导致结算死锁，或使 Harness 处于错误的阶段。

预期的规则如下：

- 当处于忙碌状态时，结构化操作保持拒绝状态。
- 队列操作在文档说明的轮次安全点被接受。
- 运行时配置设置器更新未来的快照，而不修改当前的提供者请求。
- 在忙碌时进行的会话写入会被持久化排队，并以确定的顺序刷新。
- Getter 返回最新的 Harness 配置，而非运行中的轮次快照。
- 监听器/钩子目前不接收外观（facade）；如果它们闭包引用原始 Harness 并在活动运行期间调用 `waitForIdle()` 等结算 API，可能会导致死锁。未来的外观应该暴露 `runWhenIdle()` 作为替代。

`AssistantMessageStream` 已经将提供者传输流（如 SSE 或 websocket 读取）与下游事件消费解耦。因此，Harness 可以等待监听器、扩展钩子、持久化和保存点工作，而不会阻塞提供者传输读取器或重新引入临时事件队列。生命周期代码应优先选择在 Harness 边界进行显式的异步等待序列化，而不是使用“触发即忘（fire-and-forget）”的钩子/事件结算。

最终的生命周期硬化阶段应通过广泛的监听器/钩子重入测试套件来证明这些保证。

## 错误处理

目前的划分如下：

- 低层级能力和辅助函数使用 `Result<TValue, TError>`，其中预期的失败被封装且不得抛出异常，例如 `ExecutionEnv`、文件系统/ shell 操作、shell 输出捕获、资源加载和压缩辅助函数。
- 高层级变更/编排 API（如 `Session` 和 `AgentHarness`）则采用 reject/throw 机制，而不是返回可能被忽略的原始结果。
- 公共 `AgentHarness` 失败在可行时被标准化为 `AgentHarnessError`；子系统错误作为 `cause` 保留。

Harness 事件观察已提交的状态。公共变更方法在可行的情况下，在提交前验证所需的输入和持久化，然后等待通知。如果钩子或订阅者在提交后失败，状态更改不会回滚，公共方法将以 `AgentHarnessError` 代码 `"hook"` 拒绝。

## 状态模型

Harness 将状态分为四类。

### Harness 配置 (Harness config)

Harness 配置是由应用程序或扩展设置的最新运行时配置：

- 模型 (model)
- 思考层级 (thinking level)
- 工具 (tools)
- 活动工具名称 (active tool names)
- 资源 (resources)
- 流选项 (stream options)
- 系统提示词或系统提示词提供者 (system prompt or system prompt provider)

Getter 返回的是 Harness 配置。它们不返回正在进行的提供者请求所使用的快照。

Setter 立即更新 Harness 配置，包括在轮次进行期间。更改影响下一个轮次快照，而不影响当前运行的提供者请求。

`setResources()` 接受具体资源，并在每次调用时发出 `resources_update`，包含当前和之前资源的浅拷贝。应用程序负责从磁盘或其他来源加载/重新加载资源，并应使用新值调用 `setResources()`。

`getResources()` 返回当前资源的浅拷贝。它是实时的配置读取，而不是上一个轮次的快照。

### 轮次快照 (Turn snapshot)

轮次快照是用于一次 LLM 轮次的具体状态。它由 `createTurnState()` 创建，包含：

- 已持久化的会话消息
- 已解析的资源
- 已解析的系统提示词
- 模型
- 思考层级
- 所有工具
- 活动工具
- 流选项
- 派生的会话 ID

静态选项值被直接使用。系统提示词提供者回调在每次 `createTurnState()` 调用时被调用一次。该轮次的所有逻辑都使用同一个快照。

在创建快照时，资源数组会被浅拷贝。单个技能（skill）和提示词模板（prompt-template）对象不会被深拷贝。

流选项在创建快照时会被浅拷贝。`headers` 和 `metadata` 映射会被浅拷贝；它们的值不会被深拷贝。来自 `getApiKeyAndHeaders()` 的凭据在每个提供者请求时解析，以便刷新过期的令牌，但配置的流选项和派生的会话 ID 来自当前的轮次快照。

### 会话 (Session)

会话仅包含已持久化的条目。会话读取返回已持久化的状态，不包括排队的写入。

会话存储实现必须将叶节点更改持久化为 `leaf` 条目。`setLeafId()` 不是仅内存的光标更新；它追加一个持久化条目，其 `targetId` 是活动树叶节点或根节点（`null`）。重新打开存储必须从最新的影响叶节点的持久化条目中重建当前叶节点。

### 待处理会话写入 (Pending session writes)

在操作活动期间请求的会话写入将被排队为待处理会话写入。待处理写入基于不含生成字段（`id`、`parentId`、`timestamp`）的会话条目形状。

待处理会话写入总是会被持久化。它们在保存点、操作结算以及失败清理时被刷新。

计划提供一个公共的待处理写入/会话外观 API，但尚未实现。

## 操作阶段

Harness 有明确的阶段：

```ts
type AgentHarnessPhase = "idle" | "turn" | "compaction" | "branch_summary" | "retry";
```

结构化操作要求 `phase === "idle"`，并在第一个 `await` 之前同步设置阶段：

- `prompt`
- `skill`
- `promptFromTemplate`
- `compact`
- `navigateTree`

在 Harness 非空闲时启动另一个结构化操作将以 `AgentHarnessError` 代码 `"busy"` 拒绝。

以下操作允许在轮次期间适当进行：

- `steer` (引导)
- `followUp` (追问)
- `nextTurn` (下一轮)
- `abort` (中止)
- 运行时配置设置器

阶段/结算语义仍处于临时阶段，需要完整的生命周期检查。

## 轮次执行

`prompt`、`skill` 和 `promptFromTemplate` 遵循相同的流程：

1. 断言空闲并设置阶段为 `"turn"`。
2. 使用 `createTurnState()` 创建轮次快照。
3. 从该快照派生调用文本。
4. 使用 `executeTurn()` 执行轮次。

`skill` 和 `promptFromTemplate` 从传递给轮次的同一个快照中解析其资源。它们不会单独解析资源。

`steer`、`followUp` 和 `nextTurn` 接受文本及可选图像，并在内部创建用户消息。`nextTurn` 消息将在下一次用户发起的轮次中插入到新用户消息之前。

队列模式是实时的，不进行轮次快照：

- `getSteeringMode()` / `setSteeringMode()`
- `getFollowUpMode()` / `setFollowUpMode()`

在运行期间更改队列模式会影响下一次队列消耗。队列消耗发生在安全点。

## 保存点 (Save points)

保存点发生在助手轮次及其工具结果消息完成后。

在保存点，Harness 会：

1. 在该轮次 Agent 发出的消息之后刷新待处理会话写入。
2. 如果低层级循环可能继续，则创建一个新的轮次快照。
3. 在下一个提供者请求之前应用新的上下文/模型/思考层级/流选项/会话 ID 状态。

这使得在轮次期间进行的模型、思考层级、工具、资源、流选项和系统提示词更改能够影响同一次运行中的下一轮，同时绝不修改正在进行的提供者请求。由于提供者传输读取已通过 `AssistantMessageStream` 解耦，因此可以直接等待保存点工作和钩子结算，以保持对话记录/会话顺序的确定性。循环回调在保存点不会被重新创建。

低层级循环在提供者边界将 Harness 的 `ThinkingLevel` 转换为提供者的 `reasoning`：

- `"off"` -> `undefined`
- 所有其他思考层级直接透传。

在 `agent_end` 时不需要状态刷新，除了刷新剩余的待处理会话写入并清除操作阶段。确切的 `settled` 事件时机仍在审查中。

如果系统提示词回调在启动 `prompt`、`skill` 或 `promptFromTemplate` 时抛出异常，操作将以 `AgentHarnessError` 拒绝，Harness 返回空闲状态。如果它从 `prepareNextTurn` 创建的保存点快照中抛出异常，低层级 Agent 运行将记录一条助手错误消息。

## 钩子与事件 (Hooks and events)

目标钩子系统在 [hooks.md](./hooks.md) 中描述。

摘要：

- `AgentHarness` 发出类型化的钩子事件并消费类型化的结果。
- 单个钩子实现负责注册、清理、溯源（provenance）和结果归约器（reducer）。
- 观察型和变更型钩子使用同一个特定于事件的 `on()` API；事件结果类型决定了处理器是否可以返回结果。
- 产生结果的事件由类型化的归约表处理；特定于应用的钩子仅为产生结果的应用特定事件添加归约器。
- 钩子注册溯源是注册上的侧面元数据。资源和工具溯源属于应用特定的具体值类型。
- 钩子上下文应当是一个平面的外观对象，而不是原始内部结构或延迟绑定的 Getter 迷宫。

事件负载描述正在发生的事情。Harness Getter 描述用于未来快照的最新配置。钩子和监听器结算应尽可能按生命周期顺序等待；传输背压由 `AssistantMessageStream` 在 Harness 之下处理，因此 Harness 不需要单独的异步事件队列来保持 SSE 或 websocket 读取流动。

## 计划中的会话外观 (Planned session facade)

扩展最终应通过 Harness 作用域的 `HarnessSession` 外观而非原始会话进行交互。该外观应包装内部会话并强制执行 Harness 待处理写入排序语义。一旦该外观存在，钩子和事件监听器就可以接收到一个暴露完整 `AgentHarness` 加会话外观的上下文，而不必直接访问无序的原始会话写入。

计划的读取语义：

- 读取委托给已持久化的会话状态。
- 读取不包括排队的待处理写入。

计划的写入语义：

- 空闲：立即持久化。
- 忙碌：作为待处理会话写入入队。

计划中的诊断 API 可能会显式暴露待处理写入：

```ts
getPendingWrites(): readonly PendingSessionWrite[]
```

Agent 发出的消息在 `message_end` 时持久化以保持对话顺序。待处理的扩展/会话写入在保存点的这些消息之后刷新。

## 中止 (Abort)

允许在轮次期间中止。它会中止低层级运行并清除引导/追问队列。

中止不会清除 `nextTurn` 消息。使用 `nextTurn()` 排队的消息在中止后依然存在，并将在下一次用户发起的轮次中插入到用户消息之前。

中止不会丢弃待处理会话写入。待处理写入将在下一个保存点（如果达到）、在 `agent_end` 或在操作失败清理时刷新。

中止屏障（barrier）语义仍需审计。

## 压缩与树导航 (Compaction and tree navigation)

压缩和树导航是结构化的会话变更。

它们仅在空闲时允许且不排队。它们在已持久化的会话状态上操作。下一个提示词会创建一个新的轮次快照。

分支摘要生成是树导航操作的一部分。

自动压缩和重试决策点尚未在 `AgentHarness` 中实现。

## 测试组织

Harness 测试应按领域保持专注，而不是增长为一个巨大的杂项文件。

当前结构：

- `packages/agent/test/harness/agent-harness.test.ts`: 核心生命周期和公共 API 行为。
- `packages/agent/test/harness/agent-harness-stream.test.ts`: 流选项和提供者钩子语义。

偏好的未来结构：

- `agent-harness-resources.test.ts`: 资源快照/加载语义。
- `agent-harness-tools.test.ts`: 工具注册表 Getter、活动工具语义及更新事件。
- `agent-harness-lifecycle.test.ts`: 阶段/保存点/结算/重入行为。

使用 `pi-ai` 模拟提供者（`registerFauxProvider`, `fauxAssistantMessage`）进行确定性的 Harness/提供者测试。模拟响应工厂可以检查 `StreamOptions`，调用 `options.onPayload`，并返回脚本化的助手消息，而无需真实的提供者 API 或网络访问。

Harness 覆盖率与默认的包测试运行分开配置：

```bash
npm run test:harness
npm run coverage:harness
```

`coverage:harness` 运行 `test/harness/**/*.test.ts` 并报告 `src/harness/**/*.ts` 以及它直接运行的非 Harness 运行时文件（`src/agent.ts` 和 `src/agent-loop.ts`）在 `coverage/harness` 中的覆盖率。仅类型的依赖（如 `src/types.ts`）不包括在内，因为它们没有实质性的运行时覆盖。

## 实现待办事项 (Implementation todo)

此列表跟踪在将 `AgentHarness` 视为可迁移之前的剩余工作。活动/计划项按从易到难的顺序排列。已完成项归档在底部。

### 1. 添加显式的工具注册表读取/更新语义

状态：进行中

已完成：

- 添加了 `setTools(tools, activeToolNames?)`。
- 添加了 `setActiveTools(toolNames)`。
- 无效的活动工具名称将以 `AgentHarnessError` 拒绝。
- 通过 `AgentHarness<TSkill, TPromptTemplate, TTool>` 添加了通用的应用工具形状。
- 从核心类型导出了 `QueueMode`。
- 添加了 `AgentHarnessOptions.steeringMode` 和 `followUpMode`。
- 添加了实时的 `getSteeringMode()` / `setSteeringMode()` 和 `getFollowUpMode()` / `setFollowUpMode()`。
- 添加了 `getTools()` 和 `getActiveTools()`。
- 添加了 `tools_update` 可观察性事件，包括仅限活动工具的更新。
- 活动工具更改作为分支作用域的 `active_tools_change` 条目持久化。
- 重复的工具名称和重复的活动工具名称将被拒绝。

剩余：

- 无。

备注：

- 可观察性设计：[observability.md](./observability.md)

### 2. 设计每个 `AgentHarness` 的模型注册表

状态：计划中

已完成：

- 保留了当前的 `setModel()` 行为。

剩余：

- 决定应用程序如何提供模型注册表。
- 决定 Harness 存储具体的 `Model` 对象、模型引用，还是两者兼有。
- 根据注册表验证模型选择。
- 定义活动轮次和保存点期间的模型更改语义。

### 3. 完整的 `AgentHarness` 生命周期/状态检查

状态：进行中

已完成：

- 移除了构造函数中的 `void syncFromTree()`、`syncFromTree()`、`liveOperationId` 和 `shell()`。
- 添加了 `createTurnState()`、`applyTurnState()` 和 `executeTurn()`。
- 添加了显式的 `phase` 以取代布尔值空闲状态。
- 保存点刷新上下文、模型、思考层级、流选项和会话快照状态。
- 待处理会话写入使用不含生成字段的会话条目形状。
- 待处理会话写入在保存点、结算和失败清理时刷新。
- `steer`、`followUp` 和 `nextTurn` 从文本及可选图像创建用户消息。
- `nextTurn` 消息插入在新的用户提示词之前。
- 结构化的压缩/树操作使用 `finally` 恢复阶段。
- 公共 Harness 失败将子系统原因归一化为 `AgentHarnessError`。
- 待处理会话写入逐个刷新，且不会在失败时丢失。
- 如果队列更新通知失败，队列消耗将回滚。
- `message_end` 持久化发生在订阅者通知之前。
- `abort()` 在通知前发出取消信号，且在通知错误期间仍等待空闲。
- 空闲状态下的模型/思考/工具更新在提交内存状态前进行验证和持久化。
- `setLeafId()` 持久化耐用的 `leaf` 条目，以便树导航在存储重新打开后依然存在。

剩余：

- 敲定阶段/空闲语义。
- 审计 `settled` 是否可能过早触发。
- 使 `settled` 回调内部的会话写入具有确定性。
- 审计 `agent_end` 附近的追问行为。
- 实现自动压缩决策点。
- 实现重试处理。
- 针对 coding-agent 验证 `before_agent_start` 钩子语义。
- 决定 `before_agent_start` 是否需要更多轮次信息，如工具/工具片段。
- 文档化或更改忙碌时的运行时配置事件时机。
- 审计 `abort()` 屏障语义。

### 4. 实现通用钩子/事件扩展机制

状态：已在 [hooks.md](./hooks.md) 中设计，尚未实现

已完成：

- 移除了 `AgentHarnessContext`。
- 钩子仅接收事件负载。
- `emitHook(event)` 从 `event.type` 派生钩子类型。
- 提供者请求/负载钩子具有有序的转换语义。

剩余：

- 添加 `HookEvent`、`ResultOf`、带有通用来源元数据的注册选项，以及唯一的 `AgentHarnessHooks` 实现。
- 将结果链式处理从 `AgentHarness` 移动到归约函数中。
- 类型检查基础 Harness 归约器，使每个产生结果的 `AgentHarnessEvent` 都有归约语义。
- 使 `AgentHarness` 接受并暴露具体的钩子实例，通过构造函数推导特定于应用的钩子。
- 定义通过钩子上下文暴露的初始 Harness/上下文外观。
- 保留当前的提供者钩子行为，包括流选项补丁删除语义。
- 为归约器语义添加对等测试：转换链、补丁链、提前阻止/取消、清理、来源元数据以及类型化的应用特定归约器覆盖。

备注：

- 钩子设计：[hooks.md](./hooks.md)

### 5. 探索半耐用（semi-durable）Harness/会话恢复

状态：计划中

已完成：

- 编写了耐用性设计：[durable-harness.md](./durable-harness.md)

剩余：

- 决定会话是否拥有所有耐用 Harness 状态，或者大对象是否需要侧面文件。
- 为队列、待处理写入、操作、轮次、提供者请求和工具调用定义耐用条目。
- 为应用提供的工具、模型、扩展、资源、钩子和认证提供者定义恢复要求。
- 为未完成的 Agent 轮次、提供者请求、工具调用、压缩和树导航定义保守的恢复策略。
- 原型化基于归约器的会话条目恢复。
- 决定中断的操作是追加用户可见的消息还是仅内部操作条目。

备注：

- 提供者流不可恢复；恢复应从耐用边界重启或将操作标记为已中断。
- 未完成的工具调用重新尝试是不安全的，除非工具声明了幂等/重试安全行为。

### 6. 最终生命周期硬化套件

状态：计划中

已完成：

- 无。

剩余：

- 在相关事件中添加广泛的监听器/钩子重入测试。
- 从低层级生命周期事件和 Harness 事件测试运行时配置设置器。
- 测试模型、思考、资源、工具、活动工具和流选项的运行时配置可观察性。
- 在活动轮次和保存点期间测试资源/工具/模型/思考/流选项更新。
- 测试来自监听器和钩子的会话写入，包括 `settled` 写入。
- 测试来自轮次事件、工具事件和提供者钩子的队列操作。
- 测试忙碌时的拒绝结构化操作。
- 测试来自监听器/钩子的中止。
- 测试活动操作期间的 Getter 行为。
- 测试 Agent 发出的消息和待处理监听器写入的确定性顺序。
- 测试当异步监听器调用并等待 Harness API 时不发生死锁。
- 测试通过成功、提供者错误、钩子错误、中止、压缩和树导航的阶段清理。

### 7. 后续 coding-agent 迁移计划

状态：计划中

已完成：

- 无。

剩余：

- 将 coding-agent 资源映射到源加载器。
- 将应用层级的资源去重/溯源保持在 Harness 之外。
- 使扩展加载适应未来的钩子/会话外观。
- 在核心之外保留 UI/会话行为。
- 将 coding-agent 流/认证/重试/请求头行为移动到 Harness 流配置和提供者钩子。

---

## 已完成实现待办事项

### 8. 从 `AgentHarness` 中移除对 `Agent` 的依赖

状态：已完成

已完成：

- `AgentHarness` 直接调用 `runAgentLoop()`。
- Harness 拥有运行生命周期、中止控制器、队列消耗、提供者流配置、事件归约、会话持久化、待处理写入刷新和保存点快照。
- Harness 测试涵盖提示词构建、队列消耗、中止行为、保存点刷新、待处理写入排序、等待的监听器结算、工具钩子和提供者流包装。

剩余：

- 无。

备注：

- 更广泛的监听器/钩子重入覆盖在第 6 项中跟踪。

### 9. 完成精选提供者/流配置

状态：已完成

已完成：

- 添加了精选的 `AgentHarnessOptions.streamOptions`、`getStreamOptions()` 和 `setStreamOptions()`。
- 流选项、请求头、元数据和派生的会话 ID 在每个轮次快照中。
- Harness 拥有的流包装器调用 `streamSimple()`，并保持来自低层级循环的生命周期拥有的 `signal` 和 `reasoning`。
- `getApiKeyAndHeaders()` 在每个提供者请求时解析凭据。
- 实现了 `before_provider_request`、`before_provider_payload` 和 `after_provider_response` 钩子。
- 流选项补丁支持显式字段删除和有序的钩子链式处理。
- `agent-harness-stream.test.ts` 涵盖了转发、认证合并、钩子补丁/删除/链式处理、负载钩子以及忙碌/保存点快照行为。

剩余：

- 无。

### 10. 完成低层级 `Result` 清理

状态：已完成

已完成：

- 添加了通用的 `Result<TValue, TError>` 及其辅助函数。
- 更新了 `ExecutionEnv` 和 `NodeExecutionEnv` 以返回文件系统/进程操作的类型化结果。
- 拆分了文件系统和 shell 能力。
- 将 JSONL 会话存储/代码库移动到文件系统选取上，而不是直接使用 Node 导入。
- 为流式追加用例添加了 `ExecutionEnv.appendFile()`。
- 更新了技能和提示词模板加载器以消费 `ExecutionEnv` 结果。
- 更新了 shell 输出捕获以返回结果并使用 `ExecutionEnv`，包括通过 `appendFile()` 的全输出溢出（spill）。
- 从浏览器安全的根导出中移除了 `NodeExecutionEnv`。
- 在通用截断工具中使用运行时中立的 UTF-8 处理替换了 `Buffer` 的使用。
- 将压缩和分支摘要辅助函数转换为类型化的结果返回。
- 添加了 `readTextLines()`，使 JSONL 元数据加载仅读取头行。
- 在取消没有意义的地方，从 Node 文件系统方法中移除了无用的中止处理。
- 将跨越会话边界的文件系统错误映射到类型化的 `SessionError`。
- 添加了类型化的分支摘要错误和具有因果关系的公共 Harness 错误规范化。
- 资源加载器报告非 `not_found` 文件系统失败的结构化诊断信息。
- 扩展了 `NodeExecutionEnv` 对文件操作、执行错误、中止、回调、超时和 shell 输出溢出的测试。

剩余：

- 无。

备注：

- 保持返回 `Result` 的低层级能力/辅助 API 不抛出异常。
- 保持会话存储/代码库/会话 API 抛出类型化的 `SessionError`。
- 保持公共的结构化 Harness 失败归一化为 `AgentHarnessError`。
- 保持 Node 特定 API 隔离在 `src/harness/env/nodejs.ts`、Node 后端存储/会话实现或明确的仅限 Node 入口点下。
- 随着 API 的添加，审计通用 Harness 工具是否使用了 Node 全局变量。
- 审计包导出，以便浏览器/通用导入不会拉入仅限 Node 的模块。
- 随着 API 的演进，继续扩展 `ExecutionEnv` 和 shell 输出契约测试。
