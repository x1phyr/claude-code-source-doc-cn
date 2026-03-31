# 缓存策略

## 概述

Claude Code 使用多层缓存策略优化 API 调用和响应性能，包括 Prompt Cache、文件状态缓存、会话缓存和缓存断点检测。

## 核心文件

- [src/services/api/promptCacheBreakDetection.ts](src/services/api/promptCacheBreakDetection.ts) — 缓存断点检测
- [src/services/api/claude.ts](src/services/api/claude.ts) — API 缓存逻辑
- [src/context.ts](src/context.ts) — 文件状态缓存

## 缓存类型

| 类型 | 用途 | TTL |
|------|------|-----|
| Prompt Cache | 系统 Prompt 缓存 | 5分钟 / 1小时 |
| 文件状态缓存 | 已读文件内容 | 会话期间 |
| 会话缓存 | 消息历史 | 会话期间 |
| 叠加层缓存 | 预测执行 | 临时 |

## Prompt Cache

### 缓存控制

```typescript
// 系统提示缓存控制
export type CacheControl =
  | { type: 'ephemeral' }                    // 5 分钟
  | { type: 'ephemeral'; ttl: '5m' }        // 5 分钟
  | { type: 'ephemeral'; ttl: '1h' }        // 1 小时

// 添加缓存控制到系统块
function addCacheControl(
  blocks: TextBlockParam[],
  ttl: '5m' | '1h',
): TextBlockParam[] {
  return blocks.map((block, index) => {
    if (index === blocks.length - 1) {
      return { ...block, cache_control: { type: 'ephemeral', ttl } }
    }
    return block
  })
}
```

### 缓存策略选择

```typescript
export function should1hCacheTTL(
  isUsingOverage: boolean,
  autoModeActive: boolean,
): boolean {
  // 1 小时缓存条件：
  // 1. 使用额外使用量配额
  // 2. 自动模式激活
  // 条件变化时不会断开缓存（latched session-stable）
  return isUsingOverage || autoModeActive
}
```

### Global Cache Strategy

```typescript
export type GlobalCacheStrategy =
  | 'tool_based'    // 工具定义缓存
  | 'system_prompt' // 系统提示缓存
  | 'none'          // 无缓存

// MCP 工具发现时策略变更
function updateCacheStrategy(
  mcpToolsDiscovered: boolean,
): GlobalCacheStrategy {
  if (mcpToolsDiscovered) {
    return 'tool_based'
  }
  return 'system_prompt'
}
```

## 缓存断点检测

### 状态跟踪

```typescript
type PreviousState = {
  systemHash: number           // 系统 Prompt 哈希
  toolsHash: number            // 工具定义哈希
  cacheControlHash: number     // 缓存控制哈希
  toolNames: string[]          // 工具名列表
  perToolHashes: Record<string, number>  // 每工具哈希
  systemCharCount: number      // 字符计数
  model: string                // 模型名
  fastMode: boolean            // 快速模式
  globalCacheStrategy: string  // 缓存策略
  betas: string[]              // Beta 头
  callCount: number            // 调用计数
  prevCacheReadTokens: number | null  // 上次读取 Token
  pendingChanges: PendingChanges | null  // 待处理变更
}
```

### 变更检测

```typescript
export function recordPromptState(snapshot: PromptStateSnapshot): void {
  const {
    system,
    toolSchemas,
    model,
    fastMode,
    globalCacheStrategy,
    betas,
    ...
  } = snapshot

  const strippedSystem = stripCacheControl(system)
  const strippedTools = stripCacheControl(toolSchemas)

  const systemHash = computeHash(strippedSystem)
  const toolsHash = computeHash(strippedTools)
  const cacheControlHash = computeHash(
    system.map(b => b.cache_control ?? null)
  )

  const prev = previousStateBySource.get(key)

  // 检测变更
  prev.pendingChanges = {
    systemPromptChanged: systemHash !== prev.systemHash,
    toolSchemasChanged: toolsHash !== prev.toolsHash,
    modelChanged: model !== prev.model,
    fastModeChanged: fastMode !== prev.fastMode,
    cacheControlChanged: cacheControlHash !== prev.cacheControlHash,
    globalCacheStrategyChanged: globalCacheStrategy !== prev.globalCacheStrategy,
    betasChanged: !arraysEqual(sortedBetas, prev.betas),
    addedTools: toolNames.filter(n => !prevToolSet.has(n)),
    removedTools: prev.toolNames.filter(n => !newToolSet.has(n)),
  }
}
```

### 缓存断点判断

```typescript
export async function checkResponseForCacheBreak(
  querySource: QuerySource,
  cacheReadTokens: number,
  cacheCreationTokens: number,
  messages: Message[],
  agentId?: AgentId,
): Promise<void> {
  const state = previousStateBySource.get(key)
  const prevCacheRead = state.prevCacheReadTokens

  // 更新状态
  state.prevCacheReadTokens = cacheReadTokens

  // 跳过首次调用
  if (prevCacheRead === null) return

  // 计算下降
  const tokenDrop = prevCacheRead - cacheReadTokens

  // 检测断点：下降 > 5% 且绝对值 > 2000 tokens
  if (
    cacheReadTokens >= prevCacheRead * 0.95 ||
    tokenDrop < MIN_CACHE_MISS_TOKENS
  ) {
    return  // 不是断点
  }

  // 构建原因解释
  const changes = state.pendingChanges
  const parts: string[] = []

  if (changes.modelChanged) {
    parts.push(`model changed (${changes.previousModel} → ${changes.newModel})`)
  }
  if (changes.systemPromptChanged) {
    parts.push(`system prompt changed`)
  }
  if (changes.toolSchemasChanged) {
    parts.push(`tools changed (+${changes.addedToolCount}/-${changes.removedToolCount})`)
  }

  // TTL 过期检测
  const timeSinceLastMsg = Date.now() - lastAssistantMessageTimestamp
  if (timeSinceLastMsg > CACHE_TTL_1HOUR_MS) {
    parts.push('possible 1h TTL expiry')
  } else if (timeSinceLastMsg > CACHE_TTL_5MIN_MS) {
    parts.push('possible 5min TTL expiry')
  }

  logEvent('tengu_prompt_cache_break', {
    systemPromptChanged: changes?.systemPromptChanged ?? false,
    toolSchemasChanged: changes?.toolSchemasChanged ?? false,
    prevCacheReadTokens: prevCacheRead,
    cacheReadTokens,
    ...
  })
}
```

## 文件状态缓存

### 缓存结构

```typescript
export type FileStateCache = Map<string, FileState>

export type FileState = {
  content: string           // 文件内容
  hash: string             // 内容哈希
  timestamp: number        // 读取时间
  source: 'read' | 'edit'  // 来源
}

// 在 ToolUseContext 中
export interface ToolUseContext {
  readFileState: FileStateCache
  ...
}
```

### 缓存使用

```typescript
// 读取文件时更新缓存
async function readFileWithCache(
  path: string,
  context: ToolUseContext,
): Promise<string> {
  const cached = context.readFileState.get(path)
  if (cached) {
    // 检查是否过期
    const stat = await statFile(path)
    if (stat.mtime <= cached.timestamp) {
      return cached.content  // 使用缓存
    }
  }

  // 读取新内容
  const content = await readFile(path)
  context.readFileState.set(path, {
    content,
    hash: computeHash(content),
    timestamp: Date.now(),
    source: 'read',
  })

  return content
}
```

### 缓存合并

```typescript
// Forked Agent 完成后合并缓存
function mergeFileStateCaches(
  parent: FileStateCache,
  child: FileStateCache,
): FileStateCache {
  const merged = new Map(parent)
  for (const [key, value] of child) {
    // 子缓存优先（更新的内容）
    merged.set(key, value)
  }
  return merged
}
```

## 会话缓存

### 消息持久化

```typescript
// 会话消息存储
export async function saveSessionMessages(
  sessionId: string,
  messages: Message[],
): Promise<void> {
  const sessionDir = getSessionDir(sessionId)
  for (const message of messages) {
    await appendMessage(sessionDir, message)
  }
}

// 加载会话历史
export async function loadSessionMessages(
  sessionId: string,
): Promise<Message[]> {
  const sessionDir = getSessionDir(sessionId)
  const files = await readdir(sessionDir)

  const messages: Message[] = []
  for (const file of files.sort()) {
    const content = await readFile(join(sessionDir, file))
    messages.push(JSON.parse(content))
  }

  return messages
}
```

## 叠加层缓存

### 预测执行缓存

```typescript
// 预测执行使用独立缓存
export async function startSpeculation(
  suggestion: string,
  context: ToolUseContext,
): Promise<ActiveSpeculationState> {
  const overlayPath = getOverlayPath(id)

  // 创建叠加层文件系统
  await createOverlayFilesystem(overlayPath)

  // 独立的文件状态缓存
  const overlayCache = new Map<string, FileState>()

  const result = await runForkedAgent({
    promptMessages: [...],
    cacheSafeParams: {
      overlayPath,
      parentSessionId: context.sessionId,
    },
    ...
  })

  return {
    status: 'active',
    overlayPath,
    fileStateCache: overlayCache,
    ...
  }
}
```

### 接受时合并

```typescript
export async function acceptSpeculation(
  state: ActiveSpeculationState,
  context: ToolUseContext,
): Promise<void> {
  // 合并文件到主文件系统
  await copyOverlayToMain(state.overlayPath, writtenPaths, cwd)

  // 合并文件状态缓存
  for (const [key, value] of state.fileStateCache) {
    context.readFileState.set(key, value)
  }

  // 清理叠加层
  safeRemoveOverlay(state.overlayPath)
}
```

## 缓存通知

### Compact 通知

```typescript
// Compact 完成后重置缓存基线
export function notifyCompaction(
  querySource: QuerySource,
  agentId?: AgentId,
): void {
  const state = previousStateBySource.get(key)
  if (state) {
    state.prevCacheReadTokens = null  // 重置基线
  }
}
```

### Cache Deletion 通知

```typescript
// Cached Microcompact 发送删除时
export function notifyCacheDeletion(
  querySource: QuerySource,
  agentId?: AgentId,
): void {
  const state = previousStateBySource.get(key)
  if (state) {
    state.cacheDeletionsPending = true
  }
}
```

## 缓存清理

### Agent 清理

```typescript
export function cleanupAgentTracking(agentId: AgentId): void {
  previousStateBySource.delete(agentId)
}
```

### 全局重置

```typescript
export function resetPromptCacheBreakDetection(): void {
  previousStateBySource.clear()
}
```

## 缓存统计

### Token 统计

```typescript
// API 响应中的缓存 Token
interface Usage {
  cache_read_input_tokens?: number      // 从缓存读取
  cache_creation_input_tokens?: number  // 创建缓存
  input_tokens: number                  // 输入 Token
  output_tokens: number                 // 输出 Token
}

// 计算缓存节省
function calculateCacheSavings(usage: Usage): number {
  const cachedTokens = usage.cache_read_input_tokens ?? 0
  // 缓存读取比普通读取便宜 90%
  return cachedTokens * 0.9
}
```

### Diff 输出

```typescript
// 缓存断点时输出 Diff
async function writeCacheBreakDiff(
  prevContent: string,
  newContent: string,
): Promise<string> {
  const diffPath = getCacheBreakDiffPath()
  const patch = createPatch('prompt-state', prevContent, newContent)
  await writeFile(diffPath, patch)
  return diffPath
}
```

## 统计日志

```typescript
logEvent('tengu_prompt_cache_break', {
  systemPromptChanged: boolean,
  toolSchemasChanged: boolean,
  modelChanged: boolean,
  addedToolCount: number,
  removedToolCount: number,
  prevCacheReadTokens: number,
  cacheReadTokens: number,
  cacheCreationTokens: number,
  timeSinceLastAssistantMsg: number,
  requestId: string,
})

logEvent('tengu_cache_hit', {
  tokens_saved: cachedTokens,
  ttl: '5m' | '1h',
})
```

## 关键文件

- [src/services/api/promptCacheBreakDetection.ts](src/services/api/promptCacheBreakDetection.ts) — 断点检测
- [src/services/api/claude.ts](src/services/api/claude.ts) — API 缓存
- [src/context.ts](src/context.ts) — 文件状态缓存
- [src/utils/sessionStorage.ts](src/utils/sessionStorage.ts) — 会话存储

## 关联文档

- [08-上下文管理/上下文压缩算法-Context-Compression-Algorithms.md](上下文压缩算法-Context-Compression-Algorithms.md)
- [08-上下文管理/Token估算机制-Token-Estimation.md](Token估算机制-Token-Estimation.md)
- [09-API与网络/API服务详解-API-Service.md](../09-API与网络/API服务详解-API-Service.md)