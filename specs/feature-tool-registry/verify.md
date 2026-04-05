# Tool Registry — 验证指南

## 改动概要

| 改动 | 文件 |
|------|------|
| 核心 `ToolRegistry` 类 | `src/tools/registry/ToolRegistry.ts` |
| 类型定义 | `src/tools/registry/types.ts` |
| 统一导出 | `src/tools/registry/index.ts` |
| 过滤逻辑整合 | `src/tools/registry/filtering.ts` |
| BuiltInToolsProvider | `src/tools/registry/providers/BuiltInToolsProvider.ts` |
| McpToolsProvider | `src/tools/registry/providers/McpToolsProvider.ts` |
| PluginToolsProvider (stub) | `src/tools/registry/providers/PluginToolsProvider.ts` |
| UserToolsProvider (stub) | `src/tools/registry/providers/UserToolsProvider.ts` |
| `tools.ts` 委托到 registry | `src/tools.ts` |
| 调用方迁移 (10 文件) | `src/cli/print.ts`, `src/hooks/useMergedTools.ts`, `src/hooks/useInboxPoller.ts`, `src/services/tools/toolExecution.ts`, `src/tools/AgentTool/resumeAgent.ts`, `src/screens/REPL.tsx`, `src/main.tsx`, `src/entrypoints/mcp.ts`, `src/utils/api.ts`, `src/utils/permissions/permissionSetup.ts` |
| 测试 | `src/tools/registry/__tests__/ToolRegistry.test.ts`, `src/tools/registry/providers/__tests__/BuiltInToolsProvider.test.ts` |

## 1. 单元测试

```bash
# 全量测试（2157 pass, 0 fail）
bun test

# 仅 registry 新增测试
bun test src/tools/registry/__tests__/ToolRegistry.test.ts
bun test src/tools/registry/providers/__tests__/BuiltInToolsProvider.test.ts

# 原有 tools 测试（确认未回归）
bun test src/__tests__/tools.test.ts
bun test tests/integration/tool-chain.test.ts
```

预期：全部通过，无新增失败。

## 2. 工具发现一致性验证

```bash
# 验证 getAllBaseTools() 与 BuiltInToolsProvider.discover() 输出一致
bun -e "
const { getAllBaseTools, getToolRegistry } = require('./src/tools.ts');
const old = getAllBaseTools();
const registry = getToolRegistry();
const builtin = registry.getByCategory('builtin');
const oldNames = old.map(t => t.name).sort();
const newNames = builtin.map(t => t.name).sort();
console.log('getAllBaseTools():', old.length, 'tools');
console.log('registry.getByCategory(builtin):', builtin.length, 'tools');
console.log('Match:', JSON.stringify(oldNames) === JSON.stringify(newNames) ? 'YES' : 'NO');
if (JSON.stringify(oldNames) !== JSON.stringify(newNames)) {
  console.log('Only in old:', oldNames.filter(n => !newNames.includes(n)));
  console.log('Only in new:', newNames.filter(n => !oldNames.includes(n)));
}
"
```

预期：两个路径输出完全相同的工具列表（名称和数量一致）。

## 3. Registry API 验证

```bash
bun -e "
const { getToolRegistry } = require('./src/tools.ts');
const registry = getToolRegistry();

// 基础 API
console.log('size:', registry.size);
console.log('has(Bash):', registry.has('Bash'));
console.log('has(Read):', registry.has('Read'));
console.log('get(Bash):', registry.get('Bash')?.name);
console.log('get(nonexistent):', registry.get('nonexistent'));

// 类别过滤
console.log('builtin:', registry.getByCategory('builtin').length);
console.log('mcp:', registry.getByCategory('mcp').length);
console.log('plugin:', registry.getByCategory('plugin').length);
console.log('user:', registry.getByCategory('user').length);

// assemblePool
const emptyCtx = {
  mode: 'default',
  additionalWorkingDirectories: new Map(),
  alwaysAllowRules: {},
  alwaysDenyRules: {},
  alwaysAskRules: {},
  isBypassPermissionsModeAvailable: false,
};
const pool = registry.assemblePool(emptyCtx, []);
console.log('assemblePool():', pool.length, 'tools');

// 排序验证（prompt-cache 稳定性）
const names = pool.map(t => t.name);
const sorted = [...names].sort((a, b) => a.localeCompare(b));
console.log('sorted correctly:', JSON.stringify(names) === JSON.stringify(sorted) ? 'YES' : 'NO');
"
```

预期：
- `size` 等于当前环境工具总数（测试环境约 28 个）
- `has/get` 对已知工具返回正确结果
- `assemblePool` 排序与 `localeCompare` 一致

## 4. 交互式 REPL 验证

```bash
bun run dev
```

交互操作：
1. 发送 `say hello` — 验证正常流式响应
2. 发送 `create a file called /tmp/test-registry.txt with content "hello"` — 验证 tool call 正常（Write 工具）
3. 发送 `read the file /tmp/test-registry.txt` — 验证 Read 工具正常
4. 发送 `edit /tmp/test-registry.txt to say "world"` — 验证 Edit 工具正常
5. 发送 `search for "hello" in /tmp/` — 验证 Grep 工具正常
6. 发送 `list files in /tmp/ matching test-*` — 验证 Glob 工具正常

预期：所有工具调用行为与改动前完全一致。

## 5. MCP 工具集成验证

```bash
# 确保有 MCP server 配置
bun run dev
```

交互操作：
1. 发送 `list my MCP tools` — 验证 MCP 工具在 assembled pool 中
2. 调用一个 MCP 工具 — 验证正常执行
3. 断开/重连 MCP server — 验证工具列表动态更新

## 6. 代码结构验证

```bash
# 确认新文件存在
ls src/tools/registry/
# 预期输出: ToolRegistry.ts  __tests__/  filtering.ts  index.ts
#           providers/  types.ts

ls src/tools/registry/providers/
# 预期输出: BuiltInToolsProvider.ts  McpToolsProvider.ts
#           PluginToolsProvider.ts  UserToolsProvider.ts  __tests__/

# 确认旧文件不再被引用
grep -r "from.*utils/toolPool" src/ --include="*.ts" --include="*.tsx"
# 预期输出: 无结果（所有引用已迁移到 registry/filtering.ts）
```

## 7. Lint 检查

```bash
bun run lint
```

预期：registry 新文件无新增 lint error（已有 error 为预存的，非本次引入）。

## 8. 构建验证

```bash
bun run build
```

预期：构建成功，产物可运行。

## 回归风险点

| 场景 | 风险 | 验证方式 |
|------|------|---------|
| prompt-cache 排序变化 | 高 — 会失效所有缓存 | 对比 `assemblePool` 输出排序与改动前一致 |
| 工具缺失 | 高 — 遗漏条件分支 | `BuiltInToolsProvider.discover()` 对比旧 `getAllBaseTools()` |
| 工具查找失败 | 中 — 执行路径核心 | `toolExecution.ts` 回退查找验证 |
| MCP 动态注册 | 中 — 运行时添加/移除 | 连接/断开 MCP server 测试 |
| REPL-only 工具隐藏 | 低 — REPL 模式特有 | `CLAUDE_CODE_REPL=1` 模式下验证 |
| Simple 模式子集 | 低 — 特殊模式 | `CLAUDE_CODE_SIMPLE=1` 验证 |
| Coordinator 模式过滤 | 低 — 特殊模式 | coordinator 模式验证 |

## 调用链路（改动后）

```
getAllBaseTools()              ← 外部入口（向后兼容）
  → getToolRegistry()          ← 获取单例
    → BuiltInToolsProvider.discover()  ← 静态工具发现
    → registry.getByCategory('builtin')

getTools(permissionContext)    ← 外部入口（向后兼容）
  → getAllBaseTools()           ← 委托 registry
  → filterByDenyRules()         ← 委托 registry
  → isEnabled() 过滤
  → REPL 模式隐藏

assembleToolPool(ctx, mcp)     ← 外部入口（向后兼容）
  → getToolRegistry().assemblePool()  ← 排序 + 去重

工具查找（useInboxPoller / toolExecution）
  → getToolRegistry().get(name)  ← O(1) Map 查找
  → fallback: findToolByName()   ← O(n) 数组回退
```
