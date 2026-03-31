# `query` 模块深度分析

## 概述

`query` 目录是 Claude Code **AI 查询引擎**的辅助子模块，为上层核心文件 `src/query.ts`（约 1710 行）提供四项关键能力：**配置快照**、**依赖注入**、**停止钩子**、**Token 预算控制**。

`query.ts` 是整个 Claude Code 的主循环——它接收用户消息、调用模型 API、执行工具调用、处理上下文压缩，并以 `AsyncGenerator` 流式输出结果。`query/` 目录从这一核心循环中**提取了横切关注点**，使 `query.ts` 更聚焦于主循环逻辑。

---

## 文件总览

| 文件 | 行数 | 职责 |
|------|------|------|
| `config.ts` | 47 | 查询配置快照（会话ID、功能开关） |
| `deps.ts` | 41 | I/O 依赖注入接口与生产工厂 |
| `stopHooks.ts` | 474 | 停止钩子执行引擎（含内存提取、自动Dream、Teammate钩子） |
| `tokenBudget.ts` | 94 | Token 预算追踪与继续/停止决策 |

---

## 一、`config.ts` — 查询配置快照

### 核心思想

在 `query()` 入口处，将**运行时不可变值**一次性快照为纯数据对象 `QueryConfig`。这为未来将 `query()` 重构为纯 reducer（`(state, event, config) => state`）铺平道路。

### 类型定义

```typescript
type QueryConfig = {
  sessionId: SessionId
  gates: {
    streamingToolExecution: boolean   // 流式工具执行
    emitToolUseSummaries: boolean     // 发出工具使用摘要
    isAnt: boolean                    // 蚂蚁内部用户
    fastModeEnabled: boolean          // 快速模式
  }
}
```

### 设计要点

- **排除 `feature()` 门控**：`feature()` 是 Bun 的编译时 tree-shaking 边界，必须在代码中内联使用，不能被快照化
- **`CACHED_MAY_BE_STALE`**：Statsig 特性门控使用带缓存的版本，在单次查询中保持一致性
- **内联 `fastModeEnabled`**：不导入 `fastMode.ts`，因为它会拉入 axios、settings、auth 等重量级模块，会改变测试中的初始化顺序

### 调用位置

在 `query.ts` 的 `queryLoop()` 函数入口处调用一次：

```typescript
const config = buildQueryConfig()
```

### 依赖关系

```
config.ts
  ├── bootstrap/state.ts          → getSessionId()
  ├── services/analytics/growthbook.ts → checkStatsigFeatureGate_CACHED_MAY_BE_STALE()
  └── utils/envUtils.ts           → isEnvTruthy()
```

---

## 二、`deps.ts` — I/O 依赖注入

### 核心思想

将 `query()` 的**核心外部依赖**抽象为 `QueryDeps` 接口，支持测试时注入 mock。这替代了之前在 6-8 个测试文件中重复使用 `jest.spyOn` 按模块 mock 的模式。

### 接口定义

```typescript
type QueryDeps = {
  callModel: typeof queryModelWithStreaming    // 模型调用
  microcompact: typeof microcompactMessages    // 微压缩
  autocompact: typeof autoCompactIfNeeded      // 自动压缩
  uuid: () => string                           // UUID 生成
}
```

### 设计要点

- **`typeof fn` 保持签名同步**：使用 TypeScript 的 `typeof` 自动与真实实现保持类型一致
- **窄作用域**：只包含 4 个最常 mock 的依赖，作为模式验证（PoC），后续 PR 可扩展 `runTools`、`handleStopHooks` 等
- **零额外模块图开销**：测试文件本身已经导入 `query.ts`（它导入了所有模块），所以引入 `deps.ts` 不增加新的模块加载

### 生产工厂

```typescript
function productionDeps(): QueryDeps {
  return {
    callModel: queryModelWithStreaming,
    microcompact: microcompactMessages,
    autocompact: autoCompactIfNeeded,
    uuid: randomUUID,
  }
}
```

### 调用位置

在 `queryLoop()` 中，通过 `params.deps ?? productionDeps()` 使用，测试时可传入 mock。

### 依赖关系

```
deps.ts
  ├── crypto                       → randomUUID
  ├── services/api/claude.ts       → queryModelWithStreaming
  ├── services/compact/autoCompact.ts → autoCompactIfNeeded
  └── services/compact/microCompact.ts → microcompactMessages
```

---

## 三、`stopHooks.ts` — 停止钩子执行引擎

### 核心思想

这是 `query/` 中**最复杂**的模块（474 行），负责在每轮查询结束后执行一系列**后台钩子**。它是一个 `AsyncGenerator`，在执行钩子的同时流式输出中间消息。

### 主函数签名

```typescript
async function* handleStopHooks(
  messagesForQuery: Message[],
  assistantMessages: AssistantMessage[],
  systemPrompt: SystemPrompt,
  userContext: { [k: string]: string },
  systemContext: { [k: string]: string },
  toolUseContext: ToolUseContext,
  querySource: QuerySource,
  stopHookActive?: boolean,
): AsyncGenerator<StreamEvent | Message | ..., StopHookResult>
```

返回值 `StopHookResult`：
```typescript
type StopHookResult = {
  blockingErrors: Message[]      // 阻塞性错误（会导致循环重试）
  preventContinuation: boolean   // 是否阻止继续
}
```

### 执行流程

```
handleStopHooks()
  │
  ├─ 1. 保存缓存安全的参数快照（仅 repl_main_thread / sdk）
  │
  ├─ 2. [TEMPLATES 特性门] Job 分类器
  │     └─ classifyAndWriteState() — 60s 超时保护
  │
  ├─ 3. [非 bare 模式] 后台任务（fire-and-forget）
  │     ├─ Prompt 建议（executePromptSuggestion）
  │     ├─ 内存提取（executeExtractMemories，EXTRACT_MEMORIES 门控）
  │     └─ 自动 Dream（executeAutoDream）
  │
  ├─ 4. [CHICAGO_MCP 特性门] Computer Use 清理
  │     └─ cleanupComputerUseAfterTurn()
  │
  ├─ 5. 执行 Stop 钩子
  │     ├─ 遍历 executeStopHooks() 的生成器输出
  │     ├─ 跟踪进度消息（hookCount、hookInfos）
  │     ├─ 收集阻塞错误和非阻塞错误
  │     ├─ 检测 preventContinuation 信号
  │     └─ 中止检测
  │
  ├─ 6. 创建 Stop Hook 摘要系统消息
  │
  ├─ 7. [Teammate 模式] 额外执行
  │     ├─ TaskCompleted 钩子（遍历该 teammate 的 in_progress 任务）
  │     └─ TeammateIdle 钩子
  │
  └─ 8. 异常处理
        └─ logEvent('tengu_stop_hook_error') + 系统警告消息
```

### 设计要点

- **条件编译**：`EXTRACT_MEMORIES`、`TEMPLATES`、`CHICAGO_MCP` 均通过 `feature()` + `require()` 实现条件编译，未启用的特性在构建时被 tree-shake 掉
- **子代理隔离**：子代理不会触发某些钩子（如 `toolUseContext.agentId` 检查），避免污染主线程状态
- **超时保护**：Job 分类器使用 `Promise.race` + 60s 超时，避免挂起
- **渐进式错误处理**：区分阻塞错误（blocking → 重试循环）和非阻塞错误（通知用户）
- **并行钩子执行**：Stop 钩子内部并行执行，通过 `hookInfos` 匹配 durationMs

### 调用位置

在 `query.ts` 主循环中，模型完成响应后、Token 预算检查前调用：

```typescript
const stopHookResult = yield* handleStopHooks(...)
if (stopHookResult.preventContinuation) {
  return { reason: 'stop_hook_prevented' }
}
if (stopHookResult.blockingErrors.length > 0) {
  // 将阻塞错误注入消息列表，继续循环
  state = { messages: [..., ...stopHookResult.blockingErrors], ... }
  continue
}
```

### 依赖关系（上游）

```
stopHooks.ts
  ├── bun:bundle                         → feature()
  ├── keybindings/shortcutFormat.ts      → getShortcutDisplay()
  ├── memdir/paths.ts                    → isExtractModeActive()
  ├── services/analytics/index.ts        → logEvent()
  ├── Tool.ts                            → ToolUseContext
  ├── types/hooks.ts                     → HookProgress
  ├── types/message.ts                   → 多种 Message 类型
  ├── utils/attachments.ts               → createAttachmentMessage()
  ├── utils/debug.ts                     → logForDebugging()
  ├── utils/errors.ts                    → errorMessage()
  ├── utils/hooks.ts                     → executeStopHooks/executeTaskCompletedHooks/executeTeammateIdleHooks
  ├── utils/hooks/postSamplingHooks.ts   → REPLHookContext
  ├── utils/messages.ts                  → 多种消息创建工厂
  ├── utils/tasks.ts                     → getTaskListId(), listTasks()
  ├── utils/teammate.ts                  → getAgentName(), isTeammate()
  ├── constants/querySource.ts           → QuerySource
  ├── services/autoDream/                → executeAutoDream()
  ├── services/PromptSuggestion/         → executePromptSuggestion()
  ├── services/extractMemories/          → [EXTRACT_MEMORIES] executeExtractMemories()
  ├── jobs/classifier.ts                 → [TEMPLATES] classifyAndWriteState()
  ├── utils/computerUse/cleanup.ts       → [CHICAGO_MCP] cleanupComputerUseAfterTurn()
  └── utils/forkedAgent.ts               → createCacheSafeParams(), saveCacheSafeParams()
```

---

## 四、`tokenBudget.ts` — Token 预算控制

### 核心思想

管理**单轮查询的 Token 消耗预算**，决定在达到预算时是继续（自动注入提示消息）还是停止。这是一个纯函数模块，无 I/O 依赖。

### 类型定义

```typescript
type BudgetTracker = {
  continuationCount: number       // 已继续次数
  lastDeltaTokens: number         // 上次增量 Token 数
  lastGlobalTurnTokens: number    // 上次全局 Turn Token 数
  startedAt: number               // 开始时间戳
}

type TokenBudgetDecision =
  | { action: 'continue', nudgeMessage: string, continuationCount, pct, turnTokens, budget }
  | { action: 'stop', completionEvent: { continuationCount, pct, turnTokens, budget, diminishingReturns, durationMs } | null }
```

### 决策逻辑

```
checkTokenBudget(tracker, agentId, budget, globalTurnTokens)
  │
  ├─ 子代理 或 无预算 或 预算 ≤ 0 → stop (completionEvent: null)
  │
  ├─ 检查边际收益递减:
  │     └─ continuationCount >= 3 且连续两次增量 < 500 tokens → diminishing
  │
  ├─ 非递减 且 Token < 预算的 90%
  │     └─ → continue（注入 nudge 消息）
  │
  ├─ 递减 或 曾经继续过
  │     └─ → stop（带 completionEvent，记录指标）
  │
  └─ 其他 → stop (completionEvent: null)
```

### 关键常量

- `COMPLETION_THRESHOLD = 0.9`：达到预算 90% 时触发决策
- `DIMINISHING_THRESHOLD = 500`：增量低于 500 tokens 视为边际递减

### 调用位置

在 `query.ts` 中 Stop Hook 之后调用，受 `TOKEN_BUDGET` 特性门控：

```typescript
if (feature('TOKEN_BUDGET')) {
  const decision = checkTokenBudget(
    budgetTracker!,
    toolUseContext.agentId,
    getCurrentTurnTokenBudget(),
    getTurnOutputTokens(),
  )
  if (decision.action === 'continue') {
    // 注入 nudge 消息，继续循环
    state = { messages: [..., createUserMessage({ content: decision.nudgeMessage, isMeta: true })], ... }
    continue
  }
}
```

### 依赖关系

```
tokenBudget.ts
  └── utils/tokenBudget.ts → getBudgetContinuationMessage()
```

---

## 模块间关系

```
src/query.ts (主查询循环 ~1710 行)
  │
  ├── query/config.ts
  │     └── buildQueryConfig() → QueryConfig
  │         在 queryLoop() 入口调用一次
  │
  ├── query/deps.ts
  │     ├── productionDeps() → QueryDeps
  │     └── params.deps ?? productionDeps() 支持测试注入
  │
  ├── query/stopHooks.ts
  │     └── handleStopHooks() → AsyncGenerator<..., StopHookResult>
  │         在每轮模型响应后调用
  │
  └── query/tokenBudget.ts
        ├── createBudgetTracker() → BudgetTracker
        └── checkTokenBudget() → TokenBudgetDecision
            在 stopHooks 通过后调用
```

## 执行时序（在 queryLoop 主循环中）

```
while (true) {
  // [1] 调用模型 API → 获取 assistantMessages
  
  // [2] 上下文压缩检查（autoCompact / reactiveCompact）
  
  // [3] stopHooks → handleStopHooks()
  //     ├─ 后台任务（prompt suggestion / memory extraction / auto-dream）
  //     ├─ Stop 钩子执行
  //     ├─ Teammate 钩子（如果适用）
  //     └─ 返回 blockingErrors / preventContinuation
  //     如果 preventContinuation → return 'stop_hook_prevented'
  //     如果 blockingErrors → 注入消息，continue 循环
  
  // [4] tokenBudget → checkTokenBudget()
  //     如果 continue → 注入 nudge 消息，continue 循环
  //     如果 stop + completionEvent → 记录分析事件
  //     → return 'completed'
  
  // [5] 工具执行（Tool Orchestration）
  //     → continue 循环
}
```

## 设计模式总结

| 模式 | 应用位置 | 目的 |
|------|----------|------|
| **依赖注入** | `deps.ts` | 解耦 I/O，便于测试 mock |
| **不可变配置快照** | `config.ts` | 为纯 reducer 重构铺路 |
| **条件编译** | `stopHooks.ts` 中的 `feature()` + `require()` | 未启用特性的零成本 dead-code elimination |
| **AsyncGenerator** | `handleStopHooks()` | 流式输出中间消息，支持取消 |
| **纯函数决策** | `checkTokenBudget()` | 无副作用，易于测试 |
| **Fire-and-forget** | 内存提取、自动 Dream、Prompt 建议 | 不阻塞主循环，后台异步执行 |
| **超时保护** | Job 分类器的 `Promise.race` | 防止后台任务无限挂起 |

## 外部引用

`query/` 模块的所有消费者：

| 文件 | 引用内容 |
|------|----------|
| `src/query.ts` | 导入全部四个模块，是唯一消费者 |
| `src/QueryEngine.ts` | 间接使用（通过 `import { query } from './query.js'`） |
