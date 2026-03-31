# 类型参考

## 概述

本文档汇总 Claude Code 的核心类型定义，便于查阅和理解。

## 消息类型

### 定义位置

[src/types/message.ts](src/types/message.ts)

```typescript
type Message =
  | UserMessage
  | AssistantMessage
  | ProgressMessage
  | AttachmentMessage
  | SystemMessage
  | StreamEvent
  | TombstoneMessage
  | ToolUseSummaryMessage

type UserMessage = {
  role: 'user'
  content: string | ContentBlockParam[]
  id: string
  timestamp: number
}

type AssistantMessage = {
  role: 'assistant'
  content: ContentBlockParam[]
  id: string
  timestamp: number
  toolUse?: ToolUseBlock[]
}
```

## 工具类型

### 定义位置

[src/Tool.ts](src/Tool.ts)

```typescript
type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData
> = {
  name: string
  aliases?: string[]
  searchHint?: string

  inputSchema: Input
  inputJSONSchema?: ToolInputJSONSchema
  outputSchema?: z.ZodType<unknown>

  call(
    args: z.infer<Input>,
    context: ToolUseContext,
    canUseTool: CanUseToolFn,
    parentMessage: AssistantMessage,
    onProgress?: ToolCallProgress<P>
  ): Promise<ToolResult<Output>>

  description(input: z.infer<Input>, options: {...}): Promise<string>

  isEnabled(): boolean
  validateInput?(...): Promise<ValidationResult>
  checkPermissions(...): Promise<PermissionResult>

  isConcurrencySafe(input): boolean
  isReadOnly(input): boolean
  isDestructive?(input): boolean
  interruptBehavior?(): 'cancel' | 'block'

  isMcp?: boolean
  mcpInfo?: { serverName: string; toolName: string }

  userFacingName(input): string
  renderToolUseMessage(...): React.ReactNode
  renderToolResultMessage?(...): React.ReactNode
  // ...更多渲染方法

  maxResultSizeChars: number
  strict?: boolean
  getPath?(input): string
  toAutoClassifierInput(input): unknown
}

type ToolResult<T = unknown> = {
  output: T
  metadata?: Record<string, unknown>
}
```

## 命令类型

### 定义位置

[src/types/command.ts](src/types/command.ts)

```typescript
type Command =
  | PromptCommand
  | LocalCommand
  | LocalJSXCommand

type CommandBase = {
  name: string
  aliases?: string[]
  description: string
  availability?: ('claude-ai' | 'console')[]
  isEnabled?: () => boolean
}

type PromptCommand = CommandBase & {
  type: 'prompt'
  contentLength: number
  progressMessage?: string
  source: 'builtin' | 'plugin' | 'skills' | 'bundled' | 'mcp' | 'commands_DEPRECATED'
  loadedFrom?: string
  hasUserSpecifiedDescription?: boolean
  whenToUse?: string
  kind?: 'workflow'
  pluginInfo?: { pluginManifest: { name: string; ... } }
  getPromptForCommand(args, context): Promise<string>
}

type LocalCommand = CommandBase & {
  type: 'local'
  handler(args, context): Promise<LocalCommandResult>
}

type LocalJSXCommand = CommandBase & {
  type: 'local-jsx'
  handler(args, context): Promise<React.ReactNode>
}
```

## 权限类型

### 定义位置

[src/types/permissions.ts](src/types/permissions.ts)

```typescript
type PermissionMode =
  | 'default'
  | 'acceptEdits'
  | 'bypassPermissions'
  | 'dontAsk'
  | 'plan'
  | 'auto'

type PermissionRule = {
  source: PermissionRuleSource
  ruleBehavior: PermissionBehavior
  ruleValue: PermissionRuleValue
}

type PermissionRuleValue = {
  toolName: string
  ruleContent?: string
}

type PermissionBehavior = 'allow' | 'deny' | 'ask'

type PermissionRuleSource =
  | 'userSettings'
  | 'projectSettings'
  | 'localSettings'
  | 'flagSettings'
  | 'policySettings'
  | 'cliArg'
  | 'command'
  | 'session'

type PermissionResult<Input> =
  | { behavior: 'allow'; updatedInput?: Input }
  | { behavior: 'deny'; message: string; updatedInput?: Input }
  | { behavior: 'ask'; message: string }
  | { behavior: 'passthrough'; message: string }
```

## MCP 类型

### 定义位置

[src/services/mcp/types.ts](src/services/mcp/types.ts)

```typescript
type McpServerConfig =
  | McpStdioServerConfig
  | McpSSEServerConfig
  | McpHTTPServerConfig
  | McpWebSocketServerConfig
  | McpSdkServerConfig

type McpStdioServerConfig = {
  type?: 'stdio'
  command: string
  args?: string[]
  env?: Record<string, string>
}

type McpSSEServerConfig = {
  type: 'sse'
  url: string
  headers?: Record<string, string>
  headersHelper?: string
  oauth?: McpOAuthConfig
}

type MCPServerConnection =
  | ConnectedMCPServer
  | FailedMCPServer
  | NeedsAuthMCPServer
  | PendingMCPServer
  | DisabledMCPServer

type ConnectedMCPServer = {
  client: Client
  name: string
  type: 'connected'
  capabilities: ServerCapabilities
  serverInfo?: { name: string; version: string }
  instructions?: string
  config: ScopedMcpServerConfig
  cleanup: () => Promise<void>
}
```

## AppState 类型

### 定义位置

[src/state/AppStateStore.ts](src/state/AppStateStore.ts)

```typescript
type AppState = DeepImmutable<{
  settings: SettingsJson
  verbose: boolean
  mainLoopModel: ModelSetting
  toolPermissionContext: ToolPermissionContext
  // ...更多字段
}> & {
  tasks: { [taskId: string]: TaskState }
  agentNameRegistry: Map<string, AgentId>
  // ...更多非不可变字段
}
```

## ID 类型

### 定义位置

[src/types/ids.ts](src/types/ids.ts)

```typescript
type AgentId = string & { readonly __brand: 'AgentId' }
type TaskId = string & { readonly __brand: 'TaskId' }
type MessageId = string & { readonly __brand: 'MessageId' }
type ToolUseId = string & { readonly __brand: 'ToolUseId' }

// 品牌类型确保类型安全
export function asAgentId(id: string): AgentId {
  return id as AgentId
}
```

## 任务类型

### 定义位置

[src/tasks/types.ts](src/tasks/types.ts)

```typescript
type TaskState =
  | LocalShellTaskState
  | LocalAgentTaskState
  | InProcessTeammateTaskState
  | RemoteAgentTaskState

type BaseTaskState = {
  id: TaskId
  type: string
  status: 'running' | 'completed' | 'failed'
  createdAt: number
  output?: unknown
  error?: string
}
```

## 设置类型

### 定义位置

[src/utils/settings/types.ts](src/utils/settings/types.ts)

```typescript
type SettingsJson = {
  permissions?: PermissionSettings
  mcpServers?: Record<string, McpServerConfig>
  hooks?: HookSettings
  theme?: string
  model?: string
  // ...更多设置字段
}
```

## 工具函数类型

### DeepImmutable

```typescript
type DeepImmutable<T> = {
  readonly [P in keyof T]: DeepImmutable<T[P]>
}
```

### AnyObject

```typescript
type AnyObject = Record<string, unknown>
```

## 关键文件

- [src/types/message.ts](src/types/message.ts) — 消息类型
- [src/Tool.ts](src/Tool.ts) — 工具类型
- [src/types/command.ts](src/types/command.ts) — 命令类型
- [src/types/permissions.ts](src/types/permissions.ts) — 权限类型
- [src/services/mcp/types.ts](src/services/mcp/types.ts) — MCP 类型
- [src/state/AppStateStore.ts](src/state/AppStateStore.ts) — 状态类型

## 关联文档

- [05-工具系统](05-tool-system.md)
- [06-命令系统](06-command-system.md)
- [07-权限系统](07-permission-system.md)
- [09-状态管理](09-state-management.md)