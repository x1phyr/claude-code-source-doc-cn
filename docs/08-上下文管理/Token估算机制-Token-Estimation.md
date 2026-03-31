# Token 估算机制

## 概述

Token 估算是 Claude Code 管理上下文大小的关键技术，用于预测和监控会话的 token 使用量，触发自动压缩和限制检查。

## 核心文件

- [src/services/tokenEstimation.ts](src/services/tokenEstimation.ts) — 估算逻辑
- [src/utils/tokens.ts](src/utils/tokens.ts) — Token 工具

## 估算方法

### 粗略估算

```typescript
export function roughTokenCountEstimation(
  content: string,
  bytesPerToken: number = 4,
): number {
  return Math.round(content.length / bytesPerToken)
}
```

**默认比例**：4 字节/token（约 1 token ≈ 4 字符）

### 文件类型特定比例

```typescript
export function bytesPerTokenForFileType(fileExtension: string): number {
  switch (fileExtension) {
    case 'json':
    case 'jsonl':
    case 'jsonc':
      return 2  // JSON 更密集，更多单字符 token
    default:
      return 4
  }
}

export function roughTokenCountEstimationForFileType(
  content: string,
  fileExtension: string,
): number {
  return roughTokenCountEstimation(
    content,
    bytesPerTokenForFileType(fileExtension),
  )
}
```

## API Token 计数

### 精确计数

```typescript
export async function countMessagesTokensWithAPI(
  messages: Anthropic.Beta.Messages.BetaMessageParam[],
  tools: Anthropic.Beta.Messages.BetaToolUnion[],
): Promise<number | null> {
  const model = getMainLoopModel()
  const betas = getModelBetas(model)
  const containsThinking = hasThinkingBlocks(messages)

  // 使用 countTokens API
  const response = await anthropic.beta.messages.countTokens({
    model: normalizeModelStringForAPI(model),
    messages,
    tools,
    ...(betas.length > 0 && { betas }),
    // 如果消息包含 thinking 块，启用 thinking
    ...(containsThinking && {
      thinking: {
        type: 'enabled',
        budget_tokens: 1024,
      },
    }),
  })

  return response.input_tokens
}
```

### Haiku 回退计数

```typescript
export async function countTokensViaHaikuFallback(
  messages: Anthropic.Beta.Messages.BetaMessageParam[],
  tools: Anthropic.Beta.Messages.BetaToolUnion[],
): Promise<number | null> {
  // 使用 Haiku 进行 token 计数（更快更便宜）
  const model = getSmallFastModel()

  const response = await anthropic.beta.messages.create({
    model: normalizeModelStringForAPI(model),
    max_tokens: 1,
    messages,
    tools,
  })

  const usage = response.usage
  return usage.input_tokens + 
    (usage.cache_creation_input_tokens || 0) + 
    (usage.cache_read_input_tokens || 0)
}
```

### Bedrock Token 计数

```typescript
async function countTokensWithBedrock({
  model,
  messages,
  tools,
}: {
  model: string
  messages: Anthropic.Beta.Messages.BetaMessageParam[]
  tools: Anthropic.Beta.Messages.BetaToolUnion[]
}): Promise<number | null> {
  const client = await createBedrockRuntimeClient()
  const modelId = isFoundationModel(model)
    ? model
    : await getInferenceProfileBackingModel(model)

  const { CountTokensCommand } = await import('@aws-sdk/client-bedrock-runtime')
  const response = await client.send(new CountTokensCommand({
    modelId,
    input: {
      invokeModel: {
        body: new TextEncoder().encode(JSON.stringify(requestBody)),
      },
    },
  }))

  return response.inputTokens ?? null
}
```

## 消息估算

### 消息列表估算

```typescript
export function roughTokenCountEstimationForMessages(
  messages: readonly {
    type: string
    message?: { content?: unknown }
    attachment?: Attachment
  }[],
): number {
  let totalTokens = 0
  for (const message of messages) {
    totalTokens += roughTokenCountEstimationForMessage(message)
  }
  return totalTokens
}
```

### 单条消息估算

```typescript
export function roughTokenCountEstimationForMessage(message: {
  type: string
  message?: { content?: unknown }
  attachment?: Attachment
}): number {
  // 用户/助手消息
  if ((message.type === 'assistant' || message.type === 'user') && message.message?.content) {
    return roughTokenCountEstimationForContent(message.message.content)
  }

  // 附件消息
  if (message.type === 'attachment' && message.attachment) {
    const userMessages = normalizeAttachmentForAPI(message.attachment)
    let total = 0
    for (const userMsg of userMessages) {
      total += roughTokenCountEstimationForContent(userMsg.message.content)
    }
    return total
  }

  return 0
}
```

## 内容块估算

### 内容块类型处理

```typescript
function roughTokenCountEstimationForBlock(
  block: string | Anthropic.ContentBlock | Anthropic.ContentBlockParam,
): number {
  if (typeof block === 'string') {
    return roughTokenCountEstimation(block)
  }

  switch (block.type) {
    case 'text':
      return roughTokenCountEstimation(block.text)

    case 'image':
    case 'document':
      // 图像/文档固定估算值
      return 2000

    case 'tool_result':
      return roughTokenCountEstimationForContent(block.content)

    case 'tool_use':
      return roughTokenCountEstimation(
        block.name + JSON.stringify(block.input ?? {})
      )

    case 'thinking':
      return roughTokenCountEstimation(block.thinking)

    case 'redacted_thinking':
      return roughTokenCountEstimation(block.data)

    default:
      return roughTokenCountEstimation(JSON.stringify(block))
  }
}
```

### 图像 Token 计算

```typescript
// Claude 图像 token 计算公式
// tokens = (width px * height px) / 750
// 图像最大尺寸 2000x2000 → ~5333 tokens

// 使用保守估算值 2000，与 microCompact 一致
if (block.type === 'image' || block.type === 'document') {
  return 2000
}
```

## Thinking 块处理

### 检测 Thinking 块

```typescript
function hasThinkingBlocks(
  messages: Anthropic.Beta.Messages.BetaMessageParam[],
): boolean {
  for (const message of messages) {
    if (message.role === 'assistant' && Array.isArray(message.content)) {
      for (const block of message.content) {
        if (
          typeof block === 'object' &&
          block !== null &&
          'type' in block &&
          (block.type === 'thinking' || block.type === 'redacted_thinking')
        ) {
          return true
        }
      }
    }
  }
  return false
}
```

### Thinking 参数

```typescript
// Token 计数时 thinking 的最小值
const TOKEN_COUNT_THINKING_BUDGET = 1024
const TOKEN_COUNT_MAX_TOKENS = 2048

// API 约束: max_tokens 必须大于 thinking.budget_tokens
```

## 提供商适配

### Vertex 适配

```typescript
// Vertex 全局端点不支持 Haiku
const isVertexGlobalEndpoint =
  isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX) &&
  getVertexRegionForModel(getSmallFastModel()) === 'global'

// Vertex 对某些 beta 头敏感
const filteredBetas = getAPIProvider() === 'vertex'
  ? betas.filter(b => VERTEX_COUNT_TOKENS_ALLOWED_BETAS.has(b))
  : betas
```

### Bedrock 适配

```typescript
// Bedrock 需要基础模型 ID，不支持推理配置文件 ARN
const modelId = isFoundationModel(model)
  ? model
  : await getInferenceProfileBackingModel(model)
```

## 工具搜索字段处理

```typescript
function stripToolSearchFieldsFromMessages(
  messages: Anthropic.Beta.Messages.BetaMessageParam[],
): Anthropic.Beta.Messages.BetaMessageParam[] {
  return messages.map(message => {
    if (!Array.isArray(message.content)) return message

    const normalizedContent = message.content.map(block => {
      // 移除 tool_use 块的 'caller' 字段
      if (block.type === 'tool_use') {
        const { id, name, input } = block
        return { type: 'tool_use', id, name, input }
      }

      // 过滤 tool_result 内容中的 tool_reference 块
      if (block.type === 'tool_result' && Array.isArray(block.content)) {
        const filteredContent = block.content.filter(
          c => !isToolReferenceBlock(c)
        )
        return filteredContent.length !== block.content.length
          ? { ...block, content: filteredContent }
          : block
      }

      return block
    })

    return { ...message, content: normalizedContent }
  })
}
```

## 关键文件

- [src/services/tokenEstimation.ts](src/services/tokenEstimation.ts) — 估算逻辑
- [src/utils/tokens.ts](src/utils/tokens.ts) — Token 工具
- [src/utils/contextAnalysis.ts](src/utils/contextAnalysis.ts) — 上下文分析

## 关联文档

- [08-上下文管理/上下文压缩算法-Context-Compression-Algorithms.md](上下文压缩算法-Context-Compression-Algorithms.md)
- [14-服务层/Compact服务-Compact-Service.md](../14-服务层/Compact服务-Compact-Service.md)