# 预测执行

## 概述

预测执行（Speculation）是 Claude Code 的性能优化特性，在用户确认前预先执行可能的操作，减少响应延迟。

## 核心文件

- [src/services/PromptSuggestion/speculation.ts](src/services/PromptSuggestion/speculation.ts) — 预测执行
- [src/state/AppStateStore.ts](src/state/AppStateStore.ts) — 状态管理

## 工作原理

```
用户输入建议生成
    ↓
预测执行启动
    ├── 叠加层文件系统
    ├── 执行工具调用
    └── 收集结果
    ↓
用户确认？
├── 是 → 合并结果到主文件系统
└── 否 → 丢弃叠加层
```

## 状态管理

### 预测状态

```typescript
export type SpeculationState =
  | { status: 'idle' }
  | { status: 'active'; id: string; startTime: number; ... }
  | { status: 'completed'; result: SpeculationResult }
  | { status: 'aborted'; reason: string }

export type SpeculationResult = {
  messages: Message[]
  fileStateCache: FileStateCache
  completionBoundary: CompletionBoundary | null
}

export const IDLE_SPECULATION_STATE: SpeculationState = { status: 'idle' }
```

### 完成边界

```typescript
export type CompletionBoundary =
  | { type: 'bash'; command: string }
  | { type: 'edit'; filePath: string }
  | { type: 'denied_tool'; toolName: string; detail: string }
  | { type: 'complete' }
```

## 预测执行启动

### 启动条件

```typescript
const MAX_SPECULATION_TURNS = 20
const MAX_SPECULATION_MESSAGES = 100

const WRITE_TOOLS = new Set(['Edit', 'Write', 'NotebookEdit'])
const SAFE_READ_ONLY_TOOLS = new Set([
  'Read',
  'Glob',
  'Grep',
  'ToolSearch',
  'LSP',
  'TaskGet',
  'TaskList',
])

async function shouldStartSpeculation(
  suggestion: string,
  context: SpeculationContext,
): Promise<boolean> {
  // 检查是否启用
  if (!isSpeculationEnabled()) return false

  // 检查是否为写操作
  if (!isWriteOperation(suggestion)) return false

  // 检查资源限制
  if (context.activeSpeculationCount >= MAX_CONCURRENT_SPECULATIONS) {
    return false
  }

  return true
}
```

### 启动执行

```typescript
export async function startSpeculation(
  suggestion: string,
  context: ToolUseContext,
): Promise<ActiveSpeculationState> {
  const id = randomUUID()
  const startTime = Date.now()

  // 创建叠加层文件系统
  const overlayPath = getOverlayPath(id)
  await createOverlayFilesystem(overlayPath)

  // 创建缓存安全参数
  const cacheSafeParams = createCacheSafeParams({
    overlayPath,
    parentSessionId: context.sessionId,
  })

  // 启动 forked agent
  const result = await runForkedAgent({
    promptMessages: [createUserMessage({ content: suggestion })],
    cacheSafeParams,
    canUseTool: createSpeculationCanUseTool(overlayPath),
    querySource: 'speculation',
    forkLabel: 'speculation',
    maxTurns: MAX_SPECULATION_TURNS,
  })

  return {
    status: 'active',
    id,
    startTime,
    overlayPath,
    messages: result.messages,
    fileStateCache: result.fileStateCache,
    completionBoundary: detectCompletionBoundary(result.messages),
  }
}
```

## 叠加层文件系统

### 路径管理

```typescript
function getOverlayPath(id: string): string {
  return join(getClaudeTempDir(), 'speculation', String(process.pid), id)
}

function getWrittenFiles(overlayPath: string): Set<string> {
  const written = new Set<string>()
  // 遍历叠加层，记录写入的文件
  // ...
  return written
}
```

### 合并到主文件系统

```typescript
async function copyOverlayToMain(
  overlayPath: string,
  writtenPaths: Set<string>,
  cwd: string,
): Promise<boolean> {
  let allCopied = true
  for (const rel of writtenPaths) {
    const src = join(overlayPath, rel)
    const dest = join(cwd, rel)
    try {
      await mkdir(dirname(dest), { recursive: true })
      await copyFile(src, dest)
    } catch {
      allCopied = false
      logForDebugging(`[Speculation] Failed to copy ${rel} to main`)
    }
  }
  return allCopied
}
```

### 清理叠加层

```typescript
function safeRemoveOverlay(overlayPath: string): void {
  rm(
    overlayPath,
    { recursive: true, force: true, maxRetries: 3, retryDelay: 100 },
    () => {},
  )
}
```

## 工具限制

### 预测执行工具白名单

```typescript
function createSpeculationCanUseTool(overlayPath: string): CanUseToolFn {
  return async (tool: Tool, input: unknown) => {
    // 写工具需要叠加层检查
    if (WRITE_TOOLS.has(tool.name)) {
      return {
        behavior: 'allow',
        // 重定向到叠加层
        updatedInput: redirectPathToOverlay(input, overlayPath),
      }
    }

    // 安全只读工具直接允许
    if (SAFE_READ_ONLY_TOOLS.has(tool.name)) {
      return { behavior: 'allow' }
    }

    // 其他工具需要检查
    if (tool.isReadOnly?.(input)) {
      return { behavior: 'allow' }
    }

    // 危险工具拒绝
    return {
      behavior: 'deny',
      message: 'Tool not allowed during speculation',
      decisionReason: { type: 'other', reason: 'Speculation safety' },
    }
  }
}
```

## 接受/拒绝

### 接受预测结果

```typescript
export async function acceptSpeculation(
  state: ActiveSpeculationState,
  context: ToolUseContext,
): Promise<SpeculationAcceptMessage> {
  const { overlayPath, messages, fileStateCache, completionBoundary } = state

  // 合并文件到主文件系统
  const writtenPaths = getWrittenFiles(overlayPath)
  await copyOverlayToMain(overlayPath, writtenPaths, getCwdState())

  // 合并文件状态缓存
  const mergedCache = mergeFileStateCaches(
    context.readFileState,
    fileStateCache,
  )
  context.readFileState.clear()
  for (const [key, value] of mergedCache) {
    context.readFileState.set(key, value)
  }

  // 清理叠加层
  safeRemoveOverlay(overlayPath)

  // 记录日志
  logSpeculation(state.id, 'accepted', state.startTime, ...)

  return {
    type: 'speculation_accept',
    messages,
    writtenFiles: [...writtenPaths],
    completionBoundary,
  }
}
```

### 拒绝预测结果

```typescript
export async function abortSpeculation(
  state: ActiveSpeculationState,
  reason: string,
): Promise<void> {
  const { overlayPath, id, startTime } = state

  // 清理叠加层（不合并）
  safeRemoveOverlay(overlayPath)

  // 记录日志
  logSpeculation(id, 'aborted', startTime, ..., { abort_reason: reason })
}
```

## 完成边界检测

```typescript
function detectCompletionBoundary(
  messages: Message[],
): CompletionBoundary | null {
  // 从后向前扫描消息
  for (let i = messages.length - 1; i >= 0; i--) {
    const message = messages[i]

    // 检测 Bash 完成
    if (isBashToolResult(message)) {
      return { type: 'bash', command: extractBashCommand(message) }
    }

    // 检测编辑完成
    if (isEditToolResult(message)) {
      return { type: 'edit', filePath: extractEditPath(message) }
    }

    // 检测拒绝的工具
    if (isDeniedTool(message)) {
      return {
        type: 'denied_tool',
        toolName: extractDeniedToolName(message),
        detail: extractDenialReason(message),
      }
    }
  }

  return null
}
```

## 消息处理

### 统计工具调用

```typescript
function countToolsInMessages(messages: Message[]): number {
  const blocks = messages
    .filter(isUserMessageWithArrayContent)
    .flatMap(m => m.message.content)
    .filter(b => b.type === 'tool_result' && !b.is_error)

  return blocks.length
}
```

### 边界工具提取

```typescript
function getBoundaryTool(boundary: CompletionBoundary | null): string | undefined {
  if (!boundary) return undefined
  switch (boundary.type) {
    case 'bash':
      return 'Bash'
    case 'edit':
    case 'denied_tool':
      return boundary.toolName
    case 'complete':
      return undefined
  }
}
```

## 统计日志

```typescript
logEvent('tengu_speculation', {
  speculation_id: id,
  outcome: 'accepted' | 'aborted' | 'error',
  duration_ms: Date.now() - startTime,
  suggestion_length: suggestion.length,
  tools_executed: countToolsInMessages(messages),
  completed: boundary !== null,
  boundary_type: boundary?.type,
  boundary_tool: getBoundaryTool(boundary),
})
```

## 关键文件

- [src/services/PromptSuggestion/speculation.ts](src/services/PromptSuggestion/speculation.ts) — 预测执行
- [src/state/AppStateStore.ts](src/state/AppStateStore.ts) — 状态管理
- [src/utils/forkedAgent.ts](src/utils/forkedAgent.ts) — 分叉代理

## 关联文档

- [12-代理系统/多代理协调-Multi-Agent-Coordination.md](多代理协调-Multi-Agent-Coordination.md)
- [02-核心引擎/消息处理流程-Message-Processing-Flow.md](../02-核心引擎/消息处理流程-Message-Processing-Flow.md)