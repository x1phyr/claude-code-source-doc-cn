# 启动流程详解

## 概述

本文档详细说明 Claude Code 的启动流程，从入口点到 REPL 运行的完整链路。

## 启动链路

```
package.json: "dev" → src/bootstrap-entry.ts
    ↓
src/bootstrapMacro.ts (注入 MACRO 全局常量)
    ↓
src/entrypoints/cli.tsx (参数解析，模式分发)
    ↓
src/main.tsx (REPL 渲染器)
    ↓
交互式会话
```

## 详细流程

### 1. 入口点：src/bootstrap-entry.ts

```typescript
import { ensureBootstrapMacro } from './bootstrapMacro'

ensureBootstrapMacro()  // 注入构建时常量

await import('./entrypoints/cli.tsx')  // 动态加载 CLI
```

**职责**：
- 确保 `MACRO` 全局对象存在（包含 VERSION、BUILD_TIME 等）
- 动态导入 CLI 主入口，实现代码分割

### 2. 构建时常量：src/bootstrapMacro.ts

```typescript
type MacroConfig = {
  VERSION: string
  BUILD_TIME: string
  PACKAGE_URL: string
  NATIVE_PACKAGE_URL: string
  VERSION_CHANGELOG: string
  ISSUES_EXPLAINER: string
  FEEDBACK_CHANNEL: string
}

export function ensureBootstrapMacro(): void {
  if (!('MACRO' in globalThis)) {
    globalThis.MACRO = defaultMacro  // 使用 package.json 的版本信息
  }
}
```

**职责**：
- 在构建时注入版本号、构建时间等常量
- 提供全局访问点，避免重复读取 package.json

### 3. CLI 主入口：src/entrypoints/cli.tsx

这是启动流程的核心，负责参数解析和模式分发。

#### 快速路径（Fast Paths）

为避免加载完整 CLI，对常见命令提供快速路径：

```typescript
async function main(): Promise<void> {
  const args = process.argv.slice(2)

  // 1. --version 快速路径：零模块加载
  if (args.length === 1 && (args[0] === '--version' || args[0] === '-v')) {
    console.log(`${MACRO.VERSION} (Claude Code)`)
    return
  }

  // 2. --dump-system-prompt：输出系统提示
  if (feature('DUMP_SYSTEM_PROMPT') && args[0] === '--dump-system-prompt') {
    // ...
    return
  }

  // 3. --claude-in-chrome-mcp：Chrome MCP 服务器
  if (process.argv[2] === '--claude-in-chrome-mcp') {
    await runClaudeInChromeMcpServer()
    return
  }

  // 4. --daemon-worker：守护进程工作器
  if (feature('DAEMON') && args[0] === '--daemon-worker') {
    await runDaemonWorker(args[1])
    return
  }

  // 5. remote-control / rc / remote / sync / bridge
  if (feature('BRIDGE_MODE') && ...) {
    await bridgeMain(args.slice(1))
    return
  }

  // 6. daemon：守护进程
  if (feature('DAEMON') && args[0] === 'daemon') {
    await daemonMain(args.slice(1))
    return
  }

  // 7. ps / logs / attach / kill：后台会话管理
  if (feature('BG_SESSIONS') && ...) {
    // 处理会话管理命令
    return
  }

  // 8. new / list / reply：模板任务
  if (feature('TEMPLATES') && ...) {
    await templatesMain(args)
    return
  }

  // 9. --worktree --tmux：工作树 + tmux
  if (hasTmuxFlag && hasWorktreeFlag) {
    // ...
  }

  // 10. --bare 模式：设置简单模式
  if (args.includes('--bare')) {
    process.env.CLAUDE_CODE_SIMPLE = '1'
  }

  // 默认路径：加载完整 CLI
  const { main: cliMain } = await import('../main.js')
  await cliMain()
}
```

#### 模式说明

| 模式 | 触发条件 | 说明 |
|------|----------|------|
| version | `--version` | 仅输出版本号 |
| dump-system-prompt | `--dump-system-prompt` | 输出系统提示（调试用） |
| chrome-mcp | `--claude-in-chrome-mcp` | 启动 Chrome MCP 服务器 |
| daemon-worker | `--daemon-worker` | 启动守护进程工作器 |
| bridge | `remote-control` 等 | 启动远程桥接 |
| daemon | `daemon` | 启动守护进程 |
| bg | `ps` 等 | 后台会话管理 |
| templates | `new` 等 | 模板任务 |
| worktree-tmux | `--worktree --tmux` | tmux 工作树 |
| interactive | 默认 | 启动交互式 REPL |

### 4. REPL 主渲染器：src/main.tsx

`main.tsx` 是一个 800KB+ 的巨大文件，包含：

- **命令行参数解析**：使用 Commander.js
- **配置初始化**：加载 settings.json
- **MCP 连接**：启动 MCP 服务器
- **状态管理**：初始化 AppState
- **REPL 渲染**：使用 Ink 渲染终端 UI

#### 主要流程

```typescript
export async function main(): Promise<void> {
  // 1. 解析命令行参数
  const program = createCommand()
  program.parse(process.argv)

  // 2. 初始化配置
  enableConfigs()

  // 3. 初始化分析
  initAnalytics()

  // 4. 连接 MCP 服务器
  await connectMcpServers()

  // 5. 初始化状态
  const appState = createAppState()

  // 6. 渲染 REPL
  render(<App appState={appState} />)
}
```

### 5. 交互式会话

用户输入后，进入查询循环：

```
用户输入 → processUserInput → QueryEngine.submitMessage → query()
    ↓
callModel() → Claude API 响应
    ↓
工具调用 → runTools() → 各工具执行
    ↓
响应生成 → UI 更新
```

## 环境变量影响

| 环境变量 | 影响 |
|----------|------|
| `CLAUDE_CODE_SIMPLE` | 简单模式，减少功能 |
| `CLAUDE_CODE_REMOTE` | 远程模式 |
| `NODE_ENV` | 环境（development/test/production） |
| `USER_TYPE` | 用户类型（ant 为内部用户） |
| `CLAUDE_CODE_DISABLE_CLAUDE_MDS` | 禁用 CLAUDE.md |

## 启动性能优化

### 快速路径设计原则

1. **零模块加载**：`--version` 不导入任何模块
2. **延迟加载**：只在需要时导入模块
3. **动态导入**：使用 `await import()` 分割代码
4. **条件编译**：使用 `feature()` 消除死代码

### 启动性能追踪

```typescript
const { profileCheckpoint } = await import('../utils/startupProfiler.js')
profileCheckpoint('cli_entry')
// ... 各阶段
profileCheckpoint('cli_after_main_complete')
```

## 关键文件

- `src/bootstrap-entry.ts` - 最顶层入口
- `src/bootstrapMacro.ts` - 构建时常量注入
- `src/entrypoints/cli.tsx` - CLI 参数解析和模式分发
- `src/main.tsx` - REPL 主渲染器
- `src/utils/startupProfiler.js` - 启动性能追踪

## 关联文档

- [01-架构总览](01-architecture-overview.md)
- [02-目录结构](02-directory-structure.md)
- [04-查询引擎](04-query-engine.md)