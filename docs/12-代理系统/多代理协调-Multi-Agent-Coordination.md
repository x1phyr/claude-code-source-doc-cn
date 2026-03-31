# 多代理协调

## 概述

多代理系统允许 Claude Code 同时运行多个独立代理，处理并行任务、分叉执行和协作工作流。

## 核心文件

- [src/tools/AgentTool/AgentTool.tsx](src/tools/AgentTool/AgentTool.tsx) — 代理工具
- [src/tasks/LocalAgentTask/LocalAgentTask.tsx](src/tasks/LocalAgentTask/LocalAgentTask.tsx) — 代理任务
- [src/utils/forkedAgent.ts](src/utils/forkedAgent.ts) — 分叉代理

## 代理类型

### 内置代理

```typescript
export type BuiltinAgentType =
  | 'claude-code-guide'    // Claude Code 指南
  | 'code-reviewer'        // 代码审查
  | 'explore'              // 代码探索
  | 'plan'                 // 规划代理

export const BUILTIN_AGENTS: Record<BuiltinAgentType, AgentDefinition> = {
  'claude-code-guide': {
    name: 'claude-code-guide',
    description: 'Claude Code 使用指南和最佳实践',
    model: 'haiku',
    systemPrompt: CLAUDE_CODE_GUIDE_PROMPT,
    tools: ['Read', 'Glob', 'Grep'],
  },
  // ...
}
```

### 自定义代理

```typescript
export interface AgentDefinition {
  name: string
  description: string
  model?: string
  systemPrompt?: string
  tools?: string[]
  maxTurns?: number
  isConcurrencySafe?: boolean
}

// 用户定义的代理
export interface UserAgentDefinition extends AgentDefinition {
  path: string  // 代理定义文件路径
}
```

## 代理执行

### Agent 工具

```typescript
export class AgentTool implements Tool {
  name = 'Agent'

  async call(
    args: AgentToolInput,
    context: ToolUseContext,
    canUseTool: CanUseToolFn,
    parentMessage: AssistantMessage,
    onProgress?: ToolCallProgress,
  ): Promise<ToolResult<AgentOutput>> {
    // 1. 解析代理定义
    const agentDef = await resolveAgentDefinition(args.subagent_type, args.prompt)

    // 2. 创建代理任务
    const task = await createAgentTask({
      definition: agentDef,
      input: args,
      parentContext: context,
    })

    // 3. 启动执行
    if (args.run_in_background) {
      // 后台执行
      const taskId = await startBackgroundTask(task)
      return { output: { taskId, status: 'running' } }
    } else {
      // 前台执行
      const result = await runAgentSync(task, onProgress)
      return { output: result }
    }
  }
}
```

### 分叉代理

```typescript
export async function runForkedAgent(options: {
  promptMessages: Message[]
  cacheSafeParams: CacheSafeParams
  canUseTool: CanUseToolFn
  querySource: QuerySource
  forkLabel: string
  maxTurns?: number
  skipCacheWrite?: boolean
  overrides?: {
    abortController?: AbortController
    model?: string
  }
}): Promise<ForkedAgentResult> {
  // 创建独立上下文
  const forkContext = createForkedContext(options)

  // 执行代理
  const messages: Message[] = []
  for await (const event of runAgentLoop(forkContext)) {
    messages.push(...event.messages)

    if (event.done) {
      break
    }
  }

  return {
    messages,
    totalUsage: aggregateUsage(messages),
    fileStateCache: forkContext.readFileState,
  }
}
```

## 任务管理

### 代理任务状态

```typescript
export type LocalAgentTaskState = {
  type: 'local_agent'
  agentId: AgentId
  description: string
  status: 'pending' | 'running' | 'completed' | 'failed'
  createdAt: number
  output?: unknown
  error?: string
  progress?: {
    summary: string
    toolCalls: number
  }
  retrieved: boolean
}

// 任务存储
export interface TaskStore {
  tasks: Record<TaskId, TaskState>
  agentNameRegistry: Map<string, AgentId>
}
```

### 后台任务

```typescript
export async function startBackgroundAgent(
  definition: AgentDefinition,
  input: AgentToolInput,
  context: ToolUseContext,
): Promise<AgentId> {
  const agentId = generateAgentId()

  // 注册任务
  const task: LocalAgentTaskState = {
    type: 'local_agent',
    agentId,
    description: input.description ?? definition.description,
    status: 'pending',
    createdAt: Date.now(),
  }

  context.getAppState().tasks[agentId] = task

  // 启动后台执行
  void runAgentInBackground(agentId, definition, input, context)

  return agentId
}

async function runAgentInBackground(
  agentId: AgentId,
  definition: AgentDefinition,
  input: AgentToolInput,
  context: ToolUseContext,
): Promise<void> {
  const task = context.getAppState().tasks[agentId] as LocalAgentTaskState
  task.status = 'running'

  try {
    const result = await runAgent(definition, input, context)
    task.output = result
    task.status = 'completed'
  } catch (error) {
    task.error = errorMessage(error)
    task.status = 'failed'
  }
}
```

## 代理协调

### 并发控制

```typescript
export const MAX_CONCURRENT_AGENTS = 5

export function canStartNewAgent(appState: AppState): boolean {
  const runningAgents = Object.values(appState.tasks)
    .filter(t => t.type === 'local_agent' && t.status === 'running')

  return runningAgents.length < MAX_CONCURRENT_AGENTS
}

export function getActiveAgents(appState: AppState): LocalAgentTaskState[] {
  return Object.values(appState.tasks)
    .filter((t): t is LocalAgentTaskState =>
      t.type === 'local_agent' && t.status === 'running'
    )
}
```

### 工具共享

```typescript
export function createAgentToolSet(
  definition: AgentDefinition,
  parentTools: Tool[],
): Tool[] {
  if (definition.tools) {
    // 使用指定的工具子集
    return definition.tools
      .map(name => parentTools.find(t => t.name === name))
      .filter((t): t is Tool => t !== undefined)
  }

  // 默认使用所有父工具
  return parentTools
}
```

### 消息隔离

```typescript
export function createForkedContext(
  parentContext: ToolUseContext,
): ToolUseContext {
  return {
    // 共享状态
    getAppState: parentContext.getAppState,

    // 隔离状态
    messages: [],
    readFileState: new Map(),
    loadedNestedMemoryPaths: new Set(),

    // 共享功能
    addNotification: parentContext.addNotification,
    setStreamMode: parentContext.setStreamMode,
    // ...
  }
}
```

## 颜色标识

```typescript
export const AGENT_COLORS = [
  'cyan',
  'magenta',
  'yellow',
  'blue',
  'green',
  'red',
] as const

export type AgentColorName = typeof AGENT_COLORS[number]

export class AgentColorManager {
  private assignedColors = new Map<AgentId, AgentColorName>()
  private colorIndex = 0

  assignColor(agentId: AgentId): AgentColorName {
    if (this.assignedColors.has(agentId)) {
      return this.assignedColors.get(agentId)!
    }

    const color = AGENT_COLORS[this.colorIndex % AGENT_COLORS.length]
    this.colorIndex++
    this.assignedColors.set(agentId, color)
    return color
  }

  releaseColor(agentId: AgentId): void {
    this.assignedColors.delete(agentId)
  }
}
```

## 结果检索

```typescript
export async function retrieveAgentResult(
  agentId: AgentId,
  context: ToolUseContext,
): Promise<unknown> {
  const task = context.getAppState().tasks[agentId] as LocalAgentTaskState

  if (!task) {
    throw new Error(`Agent ${agentId} not found`)
  }

  if (task.status === 'running') {
    throw new Error(`Agent ${agentId} is still running`)
  }

  task.retrieved = true
  return task.output
}

export function getTaskOutputPath(agentId: AgentId): string {
  return join(getClaudeTempDir(), 'tasks', `${agentId}.json`)
}
```

## 统计日志

```typescript
logEvent('tengu_agent_started', {
  agent_id: agentId,
  agent_type: definition.name,
  is_background: true,
  model: definition.model,
})

logEvent('tengu_agent_completed', {
  agent_id: agentId,
  status: 'completed' | 'failed',
  duration_ms: endTime - startTime,
  tool_calls: toolCallCount,
  tokens_used: totalTokens,
})
```

## 关键文件

- [src/tools/AgentTool/AgentTool.tsx](src/tools/AgentTool/AgentTool.tsx) — 代理工具
- [src/tasks/LocalAgentTask/LocalAgentTask.tsx](src/tasks/LocalAgentTask/LocalAgentTask.tsx) — 任务实现
- [src/utils/forkedAgent.ts](src/utils/forkedAgent.ts) — 分叉代理
- [src/tools/AgentTool/agentColorManager.ts](src/tools/AgentTool/agentColorManager.ts) — 颜色管理

## 关联文档

- [12-代理系统/预测执行-Speculation-Prediction.md](预测执行-Speculation-Prediction.md)
- [02-核心引擎/状态管理-State-Management.md](../02-核心引擎/状态管理-State-Management.md)