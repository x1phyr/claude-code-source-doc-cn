# 上下文压缩算法

## 概述

上下文压缩是 Claude Code 管理会话长度的核心机制。当会话接近上下文窗口限制时，系统自动将旧消息压缩为摘要，释放 token 空间同时保留关键信息。

## 核心文件

- [src/services/compact/compact.ts](src/services/compact/compact.ts) — 主压缩逻辑
- [src/services/compact/snipCompact.ts](src/services/compact/snipCompact.ts) — Snip 压缩
- [src/services/compact/microCompact.ts](src/services/compact/microCompact.ts) — 微压缩
- [src/services/compact/prompt.ts](src/services/compact/prompt.ts) — 压缩提示词

## 压缩类型

### 1. 完整压缩 (Full Compact)

手动或自动触发的完整会话压缩：

```typescript
export async function compactConversation(
  messages: Message[],
  context: ToolUseContext,
  cacheSafeParams: CacheSafeParams,
  suppressFollowUpQuestions: boolean,
  customInstructions?: string,
  isAutoCompact: boolean = false,
): Promise<CompactionResult>
```

**流程**：
1. 执行 PreCompact Hooks
2. 构建压缩提示词
3. 调用 API 生成摘要
4. 创建压缩边界标记
5. 生成后压缩附件（文件、技能、计划）
6. 执行 PostCompact Hooks

### 2. 部分压缩 (Partial Compact)

围绕指定消息的部分压缩：

```typescript
export async function partialCompactConversation(
  allMessages: Message[],
  pivotIndex: number,
  context: ToolUseContext,
  cacheSafeParams: CacheSafeParams,
  userFeedback?: string,
  direction: PartialCompactDirection = 'from',
): Promise<CompactionResult>
```

**方向**：
- `from`：保留 pivot 之前的消息，压缩之后的部分
- `up_to`：保留 pivot 之后的消息，压缩之前的部分

### 3. 微压缩 (Microcompact)

针对特定场景的轻量级压缩：

```typescript
// 微压缩处理小型上下文溢出
// 比完整压缩更快，但摘要更简略
```

## 压缩触发

### 自动压缩

当上下文接近阈值时自动触发：

```typescript
// autoCompact.ts 中的阈值检测
const shouldAutoCompact = (
  currentTokens: number,
  threshold: number,
  autoCompactThreshold: number
) => {
  return currentTokens >= threshold * autoCompactThreshold
}
```

### 手动压缩

用户执行 `/compact` 命令：

```typescript
// 命令触发
if (command === '/compact') {
  await compactConversation(messages, context, ...)
}
```

## 压缩算法细节

### Token 计算

```typescript
// 压缩前 token 计数
const preCompactTokenCount = tokenCountWithEstimation(messages)

// 压缩后 token 计数
const truePostCompactTokenCount = roughTokenCountEstimationForMessages([
  boundaryMarker,
  ...summaryMessages,
  ...postCompactFileAttachments,
  ...hookMessages,
])
```

### 摘要生成

```typescript
const compactPrompt = getCompactPrompt(customInstructions)
const summaryRequest = createUserMessage({
  content: compactPrompt,
})

// 调用 API 生成摘要
const summaryResponse = await streamCompactSummary({
  messages: messagesToSummarize,
  summaryRequest,
  appState,
  context,
  preCompactTokenCount,
  cacheSafeParams,
})
```

### PTL 重试机制

当压缩请求本身超出 prompt 长度限制时：

```typescript
export function truncateHeadForPTLRetry(
  messages: Message[],
  ptlResponse: AssistantMessage,
): Message[] | null {
  // 从最早的 API 轮次开始删除
  const groups = groupMessagesByApiRound(input)
  // 计算 token 差距
  const tokenGap = getPromptTooLongTokenGap(ptlResponse)
  // 删除足够的组以覆盖差距
  // ...
}
```

## 压缩结果

### CompactionResult 结构

```typescript
export interface CompactionResult {
  boundaryMarker: SystemMessage        // 压缩边界标记
  summaryMessages: UserMessage[]       // 摘要消息
  attachments: AttachmentMessage[]     // 后压缩附件
  hookResults: HookResultMessage[]     // Hook 结果
  messagesToKeep?: Message[]           // 保留的消息（部分压缩）
  userDisplayMessage?: string          // 用户显示消息
  preCompactTokenCount?: number        // 压缩前 token 数
  postCompactTokenCount?: number       // 压缩后 token 数
  truePostCompactTokenCount?: number   // 实际上下文 token 数
  compactionUsage?: ReturnType<typeof getTokenUsage>
}
```

### 压缩边界标记

```typescript
const boundaryMarker = createCompactBoundaryMessage(
  isAutoCompact ? 'auto' : 'manual',
  preCompactTokenCount ?? 0,
  messages.at(-1)?.uuid,
)
```

## 后压缩附件

### 文件附件

```typescript
export async function createPostCompactFileAttachments(
  readFileState: Record<string, { content: string; timestamp: number }>,
  toolUseContext: ToolUseContext,
  maxFiles: number,
  preservedMessages: Message[] = [],
): Promise<AttachmentMessage[]>
```

**限制**：
- 最大文件数：5
- Token 预算：50,000
- 单文件上限：5,000 tokens

### 技能附件

```typescript
export function createSkillAttachmentIfNeeded(
  agentId?: string,
): AttachmentMessage | null {
  const invokedSkills = getInvokedSkillsForAgent(agentId)
  // 按调用时间排序，截断到预算内
  // 单技能上限：5,000 tokens
  // 总预算：25,000 tokens
}
```

### 计划附件

```typescript
export function createPlanAttachmentIfNeeded(
  agentId?: AgentId,
): AttachmentMessage | null {
  const planContent = getPlan(agentId)
  if (!planContent) return null
  return createAttachmentMessage({
    type: 'plan_file_reference',
    planFilePath,
    planContent,
  })
}
```

## Hooks 集成

### PreCompact Hooks

```typescript
const hookResult = await executePreCompactHooks(
  {
    trigger: isAutoCompact ? 'auto' : 'manual',
    customInstructions: customInstructions ?? null,
  },
  context.abortController.signal,
)
```

### PostCompact Hooks

```typescript
const postCompactHookResult = await executePostCompactHooks(
  {
    trigger: isAutoCompact ? 'auto' : 'manual',
    compactSummary: summary,
  },
  context.abortController.signal,
)
```

## 缓存共享

```typescript
// 压缩时复用主会话的 prompt cache
const promptCacheSharingEnabled = getFeatureValue_CACHED_MAY_BE_STALE(
  'tengu_compact_cache_prefix',
  true,
)

if (promptCacheSharingEnabled) {
  // 使用 forked agent 复用缓存
  const result = await runForkedAgent({
    promptMessages: [summaryRequest],
    cacheSafeParams,
    canUseTool: createCompactCanUseTool(),
    querySource: 'compact',
    forkLabel: 'compact',
    maxTurns: 1,
    skipCacheWrite: true,
  })
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
  compactionTotalTokens,
})
```

## 关键文件

- [src/services/compact/compact.ts](src/services/compact/compact.ts) — 主压缩逻辑
- [src/services/compact/autoCompact.ts](src/services/compact/autoCompact.ts) — 自动压缩
- [src/services/compact/snipCompact.ts](src/services/compact/snipCompact.ts) — Snip 压缩
- [src/services/compact/microCompact.ts](src/services/compact/microCompact.ts) — 微压缩
- [src/services/compact/grouping.ts](src/services/compact/grouping.ts) — 消息分组

## 关联文档

- [08-上下文管理/Token估算机制-Token-Estimation.md](Token估算机制-Token-Estimation.md)
- [14-服务层/Compact服务-Compact-Service.md](../14-服务层/Compact服务-Compact-Service.md)
- [11-插件系统/Hook系统详解-Hooks-System.md](../11-插件系统/Hook系统详解-Hooks-System.md)