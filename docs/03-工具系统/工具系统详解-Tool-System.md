# 工具系统详解

## 概述

工具系统是 Claude Code 的核心能力，允许 Claude 执行各种操作：读写文件、运行命令、搜索代码、管理任务等。每个工具实现统一的 `Tool` 接口。

## Tool 接口

### 定义位置

`src/Tool.ts`

### 核心结构

```typescript
type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData
> = {
  // 标识
  name: string                    // 工具名称（唯一）
  aliases?: string[]              // 别名（向后兼容）
  searchHint?: string             // 搜索提示（3-10 词）

  // Schema
  inputSchema: Input              // Zod 输入 schema
  inputJSONSchema?: ToolInputJSONSchema  // JSON Schema 格式
  outputSchema?: z.ZodType<unknown>  // 输出 schema

  // 核心
  call(
    args: z.infer<Input>,
    context: ToolUseContext,
    canUseTool: CanUseToolFn,
    parentMessage: AssistantMessage,
    onProgress?: ToolCallProgress<P>
  ): Promise<ToolResult<Output>>

  description(
    input: z.infer<Input>,
    options: {...}
  ): Promise<string>

  // 生命周期
  isEnabled(): boolean            // 是否启用
  validateInput?(...): Promise<ValidationResult>  // 输入验证
  checkPermissions(...): Promise<PermissionResult>  // 权限检查

  // 特性标记
  isConcurrencySafe(input): boolean  // 是否并发安全
  isReadOnly(input): boolean         // 是否只读
  isDestructive?(input): boolean     // 是否破坏性
  interruptBehavior?(): 'cancel' | 'block'  // 中断行为

  // MCP 相关
  isMcp?: boolean                 // 是否 MCP 工具
  mcpInfo?: { serverName: string; toolName: string }  // MCP 信息

  // 延迟加载
  shouldDefer?: boolean           // 是否延迟加载
  alwaysLoad?: boolean            // 总是加载

  // UI 渲染
  userFacingName(input): string   // 用户可见名称
  renderToolUseMessage(...): React.ReactNode  // 渲染工具调用
  renderToolResultMessage?(...): React.ReactNode  // 渲染工具结果
  renderToolUseProgressMessage?(...): React.ReactNode  // 渲染进度
  renderToolUseRejectedMessage?(...): React.ReactNode  // 渲染拒绝
  renderToolUseErrorMessage?(...): React.ReactNode  // 渲染错误
  renderGroupedToolUse?(...): React.ReactNode  // 分组渲染

  // 其他
  maxResultSizeChars: number      // 结果最大字符数
  strict?: boolean                // 严格模式
  getPath?(input): string         // 获取路径
  toAutoClassifierInput(input): unknown  // 自动分类器输入
}
```

## 工具生命周期

```
┌──────────────────┐
│  isEnabled()     │  检查是否启用
└──────────────────┘
        │
        ▼
┌──────────────────┐
│  validateInput() │  验证输入参数（可选）
└──────────────────┘
        │
        ▼
┌──────────────────┐
│ checkPermissions │  检查权限
└──────────────────┘
        │
        ▼
┌──────────────────┐
│     call()       │  执行工具逻辑
└──────────────────┘
        │
        ▼
┌──────────────────┐
│   ToolResult     │  返回结果
└──────────────────┘
```

## 工具注册

### 定义位置

`src/tools.ts`

### 获取所有工具

```typescript
export function getAllBaseTools(): Tools {
  return [
    AgentTool,
    TaskOutputTool,
    BashTool,
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
    ExitPlanModeV2Tool,
    FileReadTool,
    FileEditTool,
    FileWriteTool,
    NotebookEditTool,
    WebFetchTool,
    TodoWriteTool,
    WebSearchTool,
    TaskStopTool,
    AskUserQuestionTool,
    SkillTool,
    EnterPlanModeTool,
    // ... 更多工具
  ]
}

export function getTools(permissionContext: ToolPermissionContext): Tools {
  const tools = getAllBaseTools()
  let allowedTools = filterToolsByDenyRules(tools, permissionContext)
  // ... 过滤逻辑
  return allowedTools.filter((_, i) => isEnabled[i])
}
```

## 权限检查

### PermissionResult

```typescript
type PermissionResult =
  | { behavior: 'allow'; updatedInput?: Record<string, unknown> }
  | { behavior: 'deny'; message: string; updatedInput?: Record<string, unknown> }
  | { behavior: 'ask'; message: string }
```

### canUseTool

权限检查函数，在 `src/hooks/useCanUseTool.tsx` 中实现：

```typescript
type CanUseToolFn = (
  tool: Tool,
  input: Record<string, unknown>,
  toolUseContext: ToolUseContext,
  assistantMessage: AssistantMessage,
  toolUseID: string,
  forceDecision?: boolean
) => Promise<PermissionResult>
```

## 工具分类

### 文件操作

| 工具 | 说明 | 只读 | 破坏性 |
|------|------|------|--------|
| FileReadTool | 读取文件内容 | ✓ | |
| FileEditTool | 编辑文件（差异编辑） | | |
| FileWriteTool | 写入文件 | | ✓ |
| GlobTool | 文件模式匹配 | ✓ | |
| GrepTool | 内容搜索 | ✓ | |

### 执行

| 工具 | 说明 | 并发安全 |
|------|------|----------|
| BashTool | Shell 命令执行 | 部分安全 |
| PowerShellTool | PowerShell 命令 | 部分安全 |

### 任务管理

| 工具 | 说明 |
|------|------|
| AgentTool | 启动子代理 |
| TaskCreateTool | 创建任务 |
| TaskGetTool | 获取任务 |
| TaskUpdateTool | 更新任务 |
| TaskListTool | 列出任务 |
| TaskOutputTool | 获取任务输出 |
| TaskStopTool | 停止任务 |

### 规划

| 工具 | 说明 |
|------|------|
| EnterPlanModeTool | 进入规划模式 |
| ExitPlanModeTool | 退出规划模式 |

### 工作树

| 工具 | 说明 |
|------|------|
| EnterWorktreeTool | 进入工作树 |
| ExitWorktreeTool | 退出工作树 |

### 网络

| 工具 | 说明 |
|------|------|
| WebFetchTool | 获取网页内容 |
| WebSearchTool | 搜索网页 |

### 交互

| 工具 | 说明 |
|------|------|
| AskUserQuestionTool | 询问用户 |
| TodoWriteTool | 管理待办事项 |

### 其他

| 工具 | 说明 |
|------|------|
| LSPTool | LSP 集成 |
| MCPTool | MCP 工具调用 |
| ConfigTool | 配置管理 |
| NotebookEditTool | Jupyter Notebook 编辑 |
| SkillTool | 技能调用 |

## buildTool 工厂函数

简化工具定义，提供默认实现：

```typescript
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  }
}

const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: () => false,
  isReadOnly: () => false,
  isDestructive: () => false,
  checkPermissions: (input) => Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: () => '',
  userFacingName: () => '',
}
```

## MCP 工具

MCP 服务器提供的工具动态加载：

```typescript
type Tool = {
  isMcp?: boolean
  mcpInfo?: { serverName: string; toolName: string }
}
```

## 进度报告

工具可以报告进度：

```typescript
type ToolCallProgress<P extends ToolProgressData> = (
  progress: ToolProgress<P>
) => void

type ToolProgress<P> = {
  toolUseID: string
  data: P
}
```

### 进度类型

```typescript
type ToolProgressData =
  | BashProgress        // Bash 命令进度
  | MCPProgress         // MCP 工具进度
  | AgentToolProgress   // 代理进度
  | TaskOutputProgress  // 任务输出进度
  | WebSearchProgress   // 搜索进度
  // ...
```

## 结果持久化

大结果自动持久化到文件：

```typescript
type Tool = {
  maxResultSizeChars: number  // 超过此大小则持久化
}
```

## 关键文件

- `src/Tool.ts` - Tool 类型定义
- `src/tools.ts` - 工具注册
- `src/tools/*/` - 各工具实现

## 关联文档

- [04-查询引擎](04-query-engine.md)
- [07-权限系统](07-permission-system.md)
- [10-工具清单](10-tools-catalog.md)