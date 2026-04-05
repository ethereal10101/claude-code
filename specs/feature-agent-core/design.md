# Agent Core 设计

> 来源: V6.md 四 4.11 (compaction), feature-overview Phase 3 (packages/agent), query.ts / QueryEngine.ts 抽象
> 优先级: P1
> 风险: 高

```text
┌───────────────────────────────────────────────────────────────────┐
│                    packages/agent                                 │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  AgentCore 接口                                             │  │
│  │  ├─ run(prompt, options)       → AsyncGenerator<AgentEvent>│  │
│  │  ├─ interrupt()                → void                     │  │
│  │  ├─ getMessages()              → readonly Message[]       │  │
│  │  ├─ getState()                 → AgentState               │  │
│  │  └─ setModel(model)           → void                     │  │
│  └──────────────────────┬────────────────────────────────────┘  │
│                          │                                       │
│  ┌───────────────────────┼───────────────────────────────────┐  │
│  │  AgentDeps (依赖注入)  │                                   │  │
│  │                       │                                   │  │
│  │  ┌──────────────┐ ┌───▼──────────┐ ┌───────────────────┐ │  │
│  │  │ Provider     │ │ ToolRegistry │ │ PermissionGate    │ │  │
│  │  │ (LLM 调用)   │ │ (工具执行)    │ │ (权限决策)        │ │  │
│  │  └──────────────┘ └──────────────┘ └───────────────────┘ │  │
│  │                                                       │  │  │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐│  │  │
│  │  │ OutputTarget │ │ HookLifecycle│ │ CompactionPipeline││  │  │
│  │  │ (输出渲染)   │ │ (钩子回调)    │ │ (上下文压缩)      ││  │  │
│  │  └──────────────┘ └──────────────┘ └──────────────────┘│  │  │
│  └─────────────────────────────────────────────────────────┘  │
│                          │                                       │
│  ┌───────────────────────▼───────────────────────────────────┐  │
│  │  Turn Loop (核心循环)                                      │  │
│  │                                                            │  │
│  │  1. 构建上下文 (systemPrompt + messages + context)          │  │
│  │  2. 调用 LLM (deps.provider.stream())                      │  │
│  │  3. yield AssistantMessage / StreamEvent                   │  │
│  │  4. 收集 tool_use blocks                                   │  │
│  │  5. 权限检查 (deps.permission.canUseTool())                │  │
│  │  6. 执行工具 (deps.toolRegistry.execute())                 │  │
│  │  7. yield ToolResult                                       │  │
│  │  8. 压缩检查 (deps.compaction.maybeCompact())              │  │
│  │  9. 继续 → 1                                               │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  AgentEvent (统一事件流)                                     │  │
│  │  ├─ { type: 'message', message: Message }                  │  │
│  │  ├─ { type: 'tool_start', toolUseId, toolName, input }     │  │
│  │  ├─ { type: 'tool_progress', toolUseId, progress }         │  │
│  │  ├─ { type: 'tool_result', toolUseId, result }             │  │
│  │  ├─ { type: 'permission_request', tool, result → Promise } │  │
│  │  ├─ { type: 'compaction', before, after }                  │  │
│  │  └─ { type: 'done', reason: 'end_turn' | 'max_turns' | 'interrupted' | 'error' }│
│  └─────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────┘

                         消费者
         ┌───────────────┬───────────────────┐
         │               │                   │
  ┌──────▼──────┐ ┌──────▼──────┐  ┌────────▼───────┐
  │  CLI REPL   │ │  SDK / API  │  │  Bridge / RC   │
  │  (Ink 渲染) │ │  (JSON 流)  │  │  (远程控制)    │
  └─────────────┘ └─────────────┘  └────────────────┘
         │               │                   │
         └───────────────┼───────────────────┘
                         │
                ┌────────▼────────┐
                │   Swarm 调度器   │
                │ (多 agent 协作)  │
                └─────────────────┘
```

## 当前问题

### query.ts 核心循环的耦合

`query()` 函数是整个系统的核心——消息循环、LLM 调用、工具执行、compaction 策略全在此文件中。但它与大量外部模块直接耦合：

| 耦合类别 | 当前依赖 | 耦合程度 |
|----------|---------|---------|
| API 调用 | `queryModelWithStreaming` 直接 import | 高 |
| Compaction | 5 种策略直接 import (auto, micro, snip, reactive, contextCollapse) | 高 |
| 工具执行 | `StreamingToolExecutor`, `runTools` 直接 import | 中 |
| 权限 | `CanUseToolFn` 来自 React hook | 高 |
| 全局状态 | `AppState` 通过 `toolUseContext.getAppState()` 访问 | 高 |
| Feature flags | ~10 个 `feature()` 调用控制核心路径 | 高 |
| 分析日志 | `logEvent` 散布在循环各处 | 中 |
| Session | `recordTranscript`, `sessionStorage` | 中 |

### ToolUseContext 巨型上下文

`ToolUseContext` 约 300 行类型定义、40+ 字段，横跨 UI（`setToolJSX`）、状态管理（`getAppState`/`setAppState`）、工具、权限、MCP、compaction、文件历史、归因等。这是核心循环与外部世界的单一耦合点。

### QueryEngine 会话管理耦合

`QueryEngine` 封装了 `query()`，但混入了：
- 系统 prompt 构建（`fetchSystemPromptParts`）
- SDK 消息转换（`SDKMessage` 等）
- UI 组件引用（`MessageSelector`）
- 会话持久化（`recordTranscript`）
- 插件/技能加载

### Message 类型宽泛

`Message` 类型使用 index signature + 大量 optional 字段，子类型通过 intersection + literal `type` 区分，不是严格的可辨识联合。核心循环需要频繁类型断言。

## 改动范围

### Phase 1: 定义核心接口 (纯类型，不改运行时代码)

1. **`AgentCore` 接口** — agent 的公共 API，消费者（REPL/SDK/Bridge/Swarm）只依赖此接口
2. **`AgentDeps` 接口** — 所有外部依赖的注入点，替代当前 `ToolUseContext` + `QueryDeps` 的组合
3. **`AgentEvent` 联合类型** — 统一事件流，替代当前 `StreamEvent | RequestStartEvent | Message | TombstoneMessage | ToolUseSummaryMessage` 的松散联合
4. **`AgentState` 接口** — 核心状态的严格子集（turnCount, usage, messages），替代当前对 `AppState` 的直接访问

### Phase 2: 抽象核心循环

将 `query()` 的 while-true 循环重构为 `packages/agent` 中的 `AgentLoop` 类：

```typescript
class AgentLoop {
  constructor(private deps: AgentDeps) {}

  async *run(input: AgentInput): AsyncGenerator<AgentEvent> {
    // 原始 query() 的核心循环逻辑
    // 所有外部依赖通过 thisdeps 访问
    // 不再直接 import 任何 UI / 状态 / 分析模块
  }
}
```

关键改动：
- `deps.callModel()` 替代直接 `queryModelWithStreaming` import（已有 `QueryDeps.callModel`，扩展之）
- `deps.compaction.compactIfNeeded()` 替代 5 个 compaction 策略的直接调用
- `deps.permission.canUseTool()` 替代 React hook 回调
- `deps.tools.execute()` 替代 `StreamingToolExecutor` / `runTools` 直接 import
- `deps.output.emit()` 替代散落在循环中的 side-effect
- `deps.hooks.onStop()` 替代 `handleStopHooks` 直接 import
- Feature flag 分支改为 deps 中的策略方法（如 `deps.compaction.strategies`）

### Phase 3: 重构 QueryEngine

将 `QueryEngine` 拆为两层：

```
┌──────────────────────────────────┐
│  QueryEngine (会话编排层)         │
│  - 消息持久化                     │
│  - 系统构建                       │
│  - SDK 消息转换                   │
│  - 插件/技能加载                  │
│  - 组装 AgentDeps                │
│  └──────────┬───────────────────┘
│             │ uses               │
│  ┌──────────▼───────────────────┐
│  │  AgentCore (核心循环)         │
│  │  - 纯逻辑，无 UI 依赖         │
│  │  - 可独立测试                 │
│  │  - 可独立用于 SDK/Bridge/Swarm│
│  └──────────────────────────────┘
```

### Phase 4: 提取为 packages/agent

文件结构：

```
packages/agent/
├── index.ts              # 公共导出
├── AgentCore.ts          # AgentCore 接口实现
├── AgentLoop.ts          # 核心循环 (from query.ts)
├── types.ts              # AgentDeps, AgentEvent, AgentState, AgentInput
├── deps/
│   ├── ProviderDep.ts    # LLM 调用适配器
│   ├── ToolDep.ts        # 工具执行适配器
│   ├── PermissionDep.ts  # 权限决策适配器
│   ├── OutputDep.ts      # 输出目标适配器
│   ├── HookDep.ts        # 钩子生命周期适配器
│   └── CompactionDep.ts  # 压缩策略适配器
└── __tests__/
    ├── AgentLoop.test.ts     # 纯逻辑测试，所有 deps mock
    └── integration.test.ts   # 与真实 deps 的集成测试
```

## AgentDeps 详细设计

```typescript
interface AgentDeps {
  /** LLM 提供者 — 封装 API 调用和流处理 */
  provider: {
    stream(params: ProviderStreamParams): AsyncIterable<ProviderEvent>;
    getModel(): string;
  };

  /** 工具注册表 — 查找和执行工具 */
  tools: {
    find(name: string): Tool | undefined;
    execute(tool: Tool, input: unknown, context: ToolExecContext): Promise<ToolResult>;
  };

  /** 权限门控 — 决定工具是否允许执行 */
  permission: {
    canUseTool(tool: Tool, input: unknown, context: PermissionContext): Promise<PermissionResult>;
  };

  /** 输出目标 — 渲染消息和进度 */
  output: {
    renderMessage(message: Message): void;
    renderToolProgress(toolUseId: string, progress: unknown): void;
    renderError(error: Error): void;
  };

  /** 钩子生命周期 — 执行前后回调 */
  hooks: {
    onTurnStart(state: AgentState): Promise<void>;
    onTurnEnd(state: AgentState): Promise<void>;
    onStop(messages: Message[]): Promise<StopHookResult>;
  };

  /** 上下文压缩 — 管理对话长度 */
  compaction: {
    maybeCompact(messages: Message[], tokenCount: number): Promise<CompactionResult>;
  };

  /** 系统上下文 — 提供系统 prompt 和环境信息 */
  context: {
    getSystemPrompt(): SystemPrompt;
    getUserContext(): Record<string, string>;
    getSystemContext(): Record<string, string>;
  };

  /** 会话存储 — 转录和状态持久化 */
  session: {
    recordTranscript(messages: Message[]): Promise<void>;
    getSessionId(): string;
  };
}
```

## 依赖方向

```text
packages/agent          ← 无运行时依赖 (纯核心逻辑)
packages/agent          → 依赖注入接口 (AgentDeps 中的各接口)

packages/agent 的消费者:
  src/screens/REPL.tsx     ← 注入 Ink 输出 + React 权限 UI
  src/QueryEngine.ts       ← 注入 SDK 输出 + 会话持久化
  src/bridge/              ← 注入 Bridge 输出 + 远程权限回调
  packages/swarm/          ← 注入 Silent 输出 + 自动权限策略

packages/agent 依赖的接口由以下 feature 提供:
  feature-provider         → AgentDeps.provider
  feature-tool-registry    → AgentDeps.tools
  feature-permission       → AgentDeps.permission
  feature-output-target    → AgentDeps.output
  feature-hook-lifecycle   → AgentDeps.hooks
  feature-compaction       → AgentDeps.compaction
  feature-context-pipeline → AgentDeps.context
  feature-storage          → AgentDeps.session
```

## 与其他 Feature 的关系

| Feature | 关系 | 说明 |
|---------|------|------|
| feature-provider | 上游 | 提供 `AgentDeps.provider` 实现 |
| feature-tool-registry | 上游 | 提供 `AgentDeps.tools` 实现 |
| feature-hook-lifecycle | 上游 | 提供 `AgentDeps.hooks` 实现 |
| feature-permission | 上游 | 提供 `AgentDeps.permission` 实现 |
| feature-output-target | 上游 | 提供 `AgentDeps.output` 实现 |
| feature-compaction | 上游 | 提供 `AgentDeps.compaction` 实现 |
| feature-context-pipeline | 上游 | 提供 `AgentDeps.context` 实现 |
| feature-storage | 上游 | 提供 `AgentDeps.session` 实现 |
| feature-shell | 下游 | BashTool 通过 `AgentDeps.tools` 被调用 |
| feature-swarm | 下游 | 创建多个 AgentCore 实例协调工作 |
| feature-overview | 包含 | Phase 3 的 `packages/agent` 即为此 feature |

## 风险与缓解

| 风险 | 缓解策略 |
|------|---------|
| 核心循环重构影响所有路径 | Phase 1 先定义接口不改代码；Phase 2 保留旧代码并行运行 |
| `ToolUseContext` 40+ 字段迁移困难 | 逐字段迁移到 `AgentDeps` 子接口，旧字段先 delegate |
| Feature flag 分支硬编码 | 策略模式：不同 flag 组合 = 不同 deps 实现 |
| `Message` 类型过于宽泛 | 核心层定义严格的 `CoreMessage` 联合类型，边界处转换 |
| 重构期间 CI 回归 | 每个 Phase 完成后跑全量测试，保持 `query.ts` 作为 fallback |
