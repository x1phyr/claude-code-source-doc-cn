# Compact 服务

## 概述

Compact 服务是上下文压缩的后端服务，提供自动压缩触发、压缩策略选择和压缩状态管理。

## 核心文件

- [src/services/compact/compact.ts](src/services/compact/compact.ts) — 核心逻辑
- [src/services/compact/autoCompact.ts](src/services/compact/autoCompact.ts) — 自动压缩
- [src/services/compact/microCompact.ts](src/services/compact/microCompact.ts) — 微压缩

## 服务架构

```
autoCompact.ts (触发检测)
    ↓
compact.ts (核心压缩)
    ├── snipCompact.ts (Snip 压缩)
    ├── microCompact.ts (微压缩)
    └── sessionMemoryCompact.ts (会话记忆压缩)
    ↓
prompt.ts (压缩提示词)
```

## 自动压缩

### 触发检测

```typescript
export async function shouldAutoCompact(
  messages: Message[],
  model: string,
): Promise<{
  shouldCompact: boolean
  threshold: number
  currentTokens: number
}> {
  // 计算当前 token 数
  const currentTokens = await estimateTokens(messages)

  // 获取模型上下文限制
  const contextLimit = getContextLimitForModel(model)

  // 计算阈值
  const threshold = contextLimit * getAutoCompactThreshold()

  return {
    shouldCompact: currentTokens >= threshold,
    threshold,
    currentTokens,
  }
}

function getAutoCompactThreshold(): number {
  // 默认 80% 时触发
  return getSettings().autoCompactThreshold ?? 0.8
}
```

### 自动压缩入口

```typescript
export async function autoCompactIfNeeded(
  messages: Message[],
  context: ToolUseContext,
  cacheSafeParams: CacheSafeParams,
): Promise<CompactionResult | null> {
  const { shouldCompact, threshold, currentTokens } = await shouldAutoCompact(
    messages,
    context.options.mainLoopModel,
  )

  if (!shouldCompact) {
    return null
  }

  // 记录重新压缩信息
  const recompactionInfo: RecompactionInfo = {
    isRecompactionInChain: detectRecompactionInChain(messages),
    turnsSincePreviousCompact: countTurnsSinceLastCompact(messages),
    previousCompactTurnId: findPreviousCompactTurnId(messages),
    autoCompactThreshold: threshold,
    querySource: context.options.querySource,
  }

  // 执行压缩
  return compactConversation(
    messages,
    context,
    cacheSafeParams,
    true,  // suppress follow-up questions
    undefined,  // custom instructions
    true,  // is auto compact
    recompactionInfo,
  )
}
```

## 压缩策略

### 策略选择

```typescript
export async function selectCompactStrategy(
  messages: Message[],
  context: ToolUseContext,
): Promise<'full' | 'partial' | 'micro' | 'snip'> {
  const tokenCount = await estimateTokens(messages)
  const contextLimit = getContextLimitForModel(context.options.mainLoopModel)

  // 接近限制时使用完整压缩
  if (tokenCount >= contextLimit * 0.9) {
    return 'full'
  }

  // 中等程度使用部分压缩
  if (tokenCount >= contextLimit * 0.7) {
    return 'partial'
  }

  // 轻微超限使用微压缩
  if (tokenCount >= contextLimit * 0.5) {
    return 'micro'
  }

  // 默认使用 Snip
  return 'snip'
}
```

### Snip 压缩

```typescript
// snipCompact.ts - 轻量级压缩
export async function snipCompact(
  messages: Message[],
): Promise<Message[]> {
  // 识别可删除的消息
  const snipTargets = identifySnipTargets(messages)

  // 删除并返回
  return messages.filter((m, i) => !snipTargets.has(i))
}

function identifySnipTargets(messages: Message[]): Set<number> {
  const targets = new Set<number>()

  for (let i = 0; i < messages.length; i++) {
    const message = messages[i]

    // 标记可 snip 的消息
    if (isSnippable(message)) {
      targets.add(i)
    }
  }

  return targets
}
```

### 微压缩

```typescript
// microCompact.ts - 中等程度压缩
export async function microCompact(
  messages: Message[],
  context: ToolUseContext,
): Promise<CompactionResult> {
  // 1. 识别低价值消息
  const lowValue = identifyLowValueMessages(messages)

  // 2. 生成简化摘要
  const summary = await generateMicroSummary(lowValue)

  // 3. 替换为摘要
  const compacted = replaceWithSummary(messages, lowValue, summary)

  return {
    boundaryMarker: createCompactBoundaryMessage('auto', ...),
    summaryMessages: [summary],
    attachments: [],
    hookResults: [],
  }
}
```

## 会话记忆压缩

```typescript
// sessionMemoryCompact.ts
export async function sessionMemoryCompact(
  messages: Message[],
  context: ToolUseContext,
): Promise<CompactionResult> {
  // 提取记忆相关内容
  const memories = extractMemories(messages)

  // 存储到会话记忆
  await storeMemories(memories)

  // 压缩消息
  return compactConversation(messages, context, ...)
}
```

## 压缩提示词

### 完整压缩提示词

```typescript
// prompt.ts
export function getCompactPrompt(customInstructions?: string): string {
  return `Summarize the conversation so far, preserving all important information.

${customInstructions ? `Additional instructions: ${customInstructions}` : ''}

Focus on:
1. User's goals and requirements
2. Important decisions made
3. Current state of work
4. Key files and code changes
5. Pending tasks

Be concise but comprehensive. The summary should allow continuing the conversation seamlessly.`
}
```

### 部分压缩提示词

```typescript
export function getPartialCompactPrompt(
  customInstructions: string | undefined,
  direction: PartialCompactDirection,
): string {
  const directionText = direction === 'up_to'
    ? 'messages before the selected point'
    : 'messages after the selected point'

  return `Summarize the ${directionText}, preserving context for the remaining messages.

${customInstructions ? `Additional instructions: ${customInstructions}` : ''}

Focus on information needed to understand the preserved messages.`
}
```

### 用户摘要消息

```typescript
export function getCompactUserSummaryMessage(
  summary: string,
  suppressFollowUpQuestions: boolean,
  transcriptPath?: string,
): string {
  let message = `<conversation_summary>
${summary}
</conversation_summary>`

  if (!suppressFollowUpQuestions) {
    message += `\n\nContinue the conversation naturally. If the user had pending questions or tasks, address them.`
  }

  return message
}
```

## 压缩状态

### 状态追踪

```typescript
// compactWarningState.ts
export interface CompactWarningState {
  lastCompactTime: number
  consecutiveCompacts: number
  tokenTrend: 'increasing' | 'stable' | 'decreasing'
}

let compactWarningState: CompactWarningState = {
  lastCompactTime: 0,
  consecutiveCompacts: 0,
  tokenTrend: 'stable',
}

export function updateCompactWarningState(result: CompactionResult): void {
  const now = Date.now()
  const timeSinceLastCompact = now - compactWarningState.lastCompactTime

  compactWarningState.lastCompactTime = now

  // 检测连续压缩
  if (timeSinceLastCompact < 60000) {  // 1 分钟内
    compactWarningState.consecutiveCompacts++
  } else {
    compactWarningState.consecutiveCompacts = 0
  }

  // 更新趋势
  if (result.truePostCompactTokenCount && result.preCompactTokenCount) {
    const ratio = result.truePostCompactTokenCount / result.preCompactTokenCount
    compactWarningState.tokenTrend = ratio > 0.8 ? 'increasing' : ratio < 0.5 ? 'decreasing' : 'stable'
  }
}
```

### 警告检测

```typescript
// compactWarningHook.ts
export function shouldShowCompactWarning(): boolean {
  return (
    compactWarningState.consecutiveCompacts >= 3 ||
    compactWarningState.tokenTrend === 'increasing'
  )
}

export function getCompactWarningMessage(): string {
  if (compactWarningState.consecutiveCompacts >= 3) {
    return 'Context is filling rapidly. Consider clearing old messages or starting a new session.'
  }
  if (compactWarningState.tokenTrend === 'increasing') {
    return 'Context is growing. The conversation may need restructuring.'
  }
  return ''
}
```

## 统计事件

```typescript
logEvent('tengu_compact', {
  preCompactTokenCount,
  postCompactTokenCount,
  truePostCompactTokenCount,
  autoCompactThreshold,
  willRetriggerNextTurn,
  isAutoCompact,
  querySource,
  queryChainId,
  queryDepth,
  isRecompactionInChain,
  turnsSincePreviousCompact,
  compactionInputTokens,
  compactionOutputTokens,
  compactionCacheReadTokens,
  compactionCacheCreationTokens,
})
```

## 关键文件

- [src/services/compact/compact.ts](src/services/compact/compact.ts) — 核心逻辑
- [src/services/compact/autoCompact.ts](src/services/compact/autoCompact.ts) — 自动压缩
- [src/services/compact/snipCompact.ts](src/services/compact/snipCompact.ts) — Snip 压缩
- [src/services/compact/microCompact.ts](src/services/compact/microCompact.ts) — 微压缩
- [src/services/compact/prompt.ts](src/services/compact/prompt.ts) — 提示词
- [src/services/compact/compactWarningState.ts](src/services/compact/compactWarningState.ts) — 状态追踪

## 关联文档

- [08-上下文管理/上下文压缩算法-Context-Compression-Algorithms.md](../08-上下文管理/上下文压缩算法-Context-Compression-Algorithms.md)
- [08-上下文管理/Token估算机制-Token-Estimation.md](../08-上下文管理/Token估算机制-Token-Estimation.md)