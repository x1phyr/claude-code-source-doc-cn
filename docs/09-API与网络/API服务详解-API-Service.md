# API 服务详解

## 概述

API 服务层负责与 Anthropic API 的所有通信，包括请求构建、响应处理、流式传输和错误处理。

## 核心文件

- [src/services/api/claude.ts](src/services/api/claude.ts) — API 客户端
- [src/services/api/withRetry.ts](src/services/api/withRetry.ts) — 重试逻辑
- [src/services/api/errors.ts](src/services/api/errors.ts) — 错误处理

## 客户端创建

### 客户端工厂

```typescript
export async function createAPIClient(): Promise<Anthropic> {
  // 获取认证信息
  const auth = await getAuthConfig()

  // 创建客户端
  return new Anthropic({
    apiKey: auth.apiKey,
    baseURL: auth.baseURL,
    // 自定义 fetch
    fetch: wrapFetchWithProxy(fetch),
    // 超时配置
    timeout: getTimeoutMs(),
  })
}

async function getAuthConfig() {
  // 优先使用 OAuth token
  const oauthToken = getClaudeAIOAuthTokens()
  if (oauthToken) {
    return {
      apiKey: oauthToken.accessToken,
      baseURL: CLAUDE_API_URL,
    }
  }

  // 使用 API key
  const apiKey = await getApiKey()
  if (apiKey) {
    return { apiKey }
  }

  // Bedrock
  if (process.env.CLAUDE_CODE_USE_BEDROCK) {
    return getBedrockConfig()
  }

  // Vertex
  if (process.env.CLAUDE_CODE_USE_VERTEX) {
    return getVertexConfig()
  }

  throw new Error('No API credentials available')
}
```

## 流式请求

### 主请求函数

```typescript
export async function* queryModelWithStreaming(
  options: StreamingOptions,
): AsyncGenerator<StreamEvent | AssistantMessage> {
  const client = await createAPIClient()

  // 构建请求参数
  const params: MessageCreateParams = {
    model: options.model,
    messages: options.messages,
    system: options.systemPrompt,
    tools: buildToolSchemas(options.tools),
    max_tokens: options.maxOutputTokens ?? getMaxOutputTokensForModel(options.model),
    stream: true,
  }

  // 添加 thinking 配置
  if (options.thinkingConfig.type === 'enabled') {
    params.thinking = {
      type: 'enabled',
      budget_tokens: options.thinkingConfig.budgetTokens,
    }
  }

  // 添加快速模式
  if (options.fastMode) {
    params.service_tier = 'priority'
  }

  // 发送请求
  const stream = await client.messages.stream(params, {
    signal: options.signal,
  })

  // 处理流式响应
  for await (const event of stream) {
    yield processStreamEvent(event)
  }

  // 返回最终消息
  const finalMessage = await stream.finalMessage()
  yield {
    type: 'assistant',
    content: finalMessage.content,
    id: finalMessage.id,
    timestamp: Date.now(),
    toolUse: extractToolUseBlocks(finalMessage),
  }
}
```

### 流式事件处理

```typescript
function processStreamEvent(
  event: MessageStreamEvent,
): StreamEvent | null {
  switch (event.type) {
    case 'message_start':
      return {
        type: 'message_start',
        message: event.message,
      }

    case 'content_block_start':
      return {
        type: 'content_block_start',
        index: event.index,
        content_block: event.content_block,
      }

    case 'content_block_delta':
      return {
        type: 'content_block_delta',
        index: event.index,
        delta: event.delta,
      }

    case 'content_block_stop':
      return {
        type: 'content_block_stop',
        index: event.index,
      }

    case 'message_delta':
      return {
        type: 'message_delta',
        delta: event.delta,
        usage: event.usage,
      }

    case 'message_stop':
      return {
        type: 'message_stop',
      }

    default:
      return null
  }
}
```

## 非流式请求

```typescript
export async function queryModel(
  options: NonStreamingOptions,
): Promise<AssistantMessage> {
  const client = await createAPIClient()

  const response = await client.messages.create({
    model: options.model,
    messages: options.messages,
    system: options.systemPrompt,
    tools: buildToolSchemas(options.tools),
    max_tokens: options.maxOutputTokens,
  })

  return {
    type: 'assistant',
    content: response.content,
    id: response.id,
    timestamp: Date.now(),
    toolUse: extractToolUseBlocks(response),
  }
}
```

## 请求构建

### 工具模式构建

```typescript
function buildToolSchemas(tools: Tool[]): ToolSchema[] {
  return tools.map(tool => ({
    name: tool.name,
    description: tool.description,
    input_schema: tool.inputJSONSchema ?? zodToJsonSchema(tool.inputSchema),
  }))
}
```

### 消息规范化

```typescript
function normalizeMessages(
  messages: Message[],
  tools: Tool[],
): MessageParam[] {
  return messages.map(message => {
    switch (message.type) {
      case 'user':
        return {
          role: 'user',
          content: normalizeUserContent(message.message.content),
        }

      case 'assistant':
        return {
          role: 'assistant',
          content: normalizeAssistantContent(message.message.content, tools),
        }

      default:
        throw new Error(`Unknown message type: ${message.type}`)
    }
  })
}
```

## Prompt Cache

### 缓存配置

```typescript
function buildSystemPrompt(
  basePrompt: string,
  tools: Tool[],
): SystemBlock[] {
  return [
    // 可缓存的系统提示
    { type: 'text', text: basePrompt, cache_control: { type: 'ephemeral' } },
    // 工具定义
    ...buildCachedToolBlocks(tools),
  ]
}

function buildCachedToolBlocks(tools: Tool[]): TextBlock[] {
  const toolText = formatToolsForSystem(tools)
  return [{
    type: 'text',
    text: toolText,
    cache_control: { type: 'ephemeral' },
  }]
}
```

### 缓存利用统计

```typescript
export interface CacheUsage {
  cache_creation_input_tokens: number
  cache_read_input_tokens: number
  input_tokens: number
  output_tokens: number
}

export function getTokenUsage(message: AssistantMessage): CacheUsage | null {
  const usage = message.usage
  if (!usage) return null

  return {
    cache_creation_input_tokens: usage.cache_creation_input_tokens ?? 0,
    cache_read_input_tokens: usage.cache_read_input_tokens ?? 0,
    input_tokens: usage.input_tokens,
    output_tokens: usage.output_tokens,
  }
}
```

## 模型配置

### 模型 ID

```typescript
export const MODEL_IDS = {
  OPUS: 'claude-opus-4-6-20250801',
  SONNET: 'claude-sonnet-4-6-20250801',
  HAIKU: 'claude-haiku-4-5-20251001',
}

export const MODEL_ALIASES = {
  'opus': MODEL_IDS.OPUS,
  'sonnet': MODEL_IDS.SONNET,
  'haiku': MODEL_IDS.HAIKU,
  'claude-opus-4-6': MODEL_IDS.OPUS,
  'claude-sonnet-4-6': MODEL_IDS.SONNET,
  'claude-haiku-4-5': MODEL_IDS.HAIKU,
}
```

### 输出 Token 限制

```typescript
export function getMaxOutputTokensForModel(model: string): number {
  if (model.includes('opus')) return 16000
  if (model.includes('sonnet')) return 16000
  if (model.includes('haiku')) return 8192
  return 8192  // 默认
}
```

## 多提供商支持

### Bedrock

```typescript
async function getBedrockConfig() {
  const region = process.env.AWS_REGION || 'us-east-1'
  const credentials = await getAwsCredentials()

  return {
    baseURL: `https://bedrock-runtime.${region}.amazonaws.com`,
    apiKey: '',  // Bedrock 使用 SigV4 签名
    headers: {
      'x-amz-security-token': credentials.sessionToken,
    },
    fetch: createBedrockFetch(credentials),
  }
}
```

### Vertex AI

```typescript
async function getVertexConfig() {
  const projectId = process.env.GOOGLE_CLOUD_PROJECT
  const location = process.env.GOOGLE_CLOUD_LOCATION || 'us-central1'
  const accessToken = await getGcpAccessToken()

  return {
    baseURL: `https://${location}-aiplatform.googleapis.com/v1/projects/${projectId}/locations/${location}/publishers/anthropic/models`,
    apiKey: accessToken,
  }
}
```

## 错误处理

### 错误类型

```typescript
export class APIError extends Error {
  status?: number
  headers?: Headers
  error?: { type: string; message: string }
}

export class APIConnectionError extends Error {
  code?: string
}

export class APIUserAbortError extends Error {}
```

### 错误消息

```typescript
export const PROMPT_TOO_LONG_ERROR_MESSAGE = 'API Error - Prompt is too long:'
export const REPEATED_529_ERROR_MESSAGE = 'API is overloaded. Please wait a minute and try again.'

export function startsWithApiErrorPrefix(text: string): boolean {
  return text.startsWith('API Error -')
}

export function getPromptTooLongTokenGap(response: AssistantMessage): number | undefined {
  const match = response.message.content.match(/exceed context limit: (\d+) \+ (\d+) > (\d+)/)
  if (match) {
    return parseInt(match[1]) + parseInt(match[2]) - parseInt(match[3])
  }
}
```

## 统计日志

```typescript
logEvent('tengu_api_request', {
  model: options.model,
  input_tokens: usage.input_tokens,
  cache_read_tokens: usage.cache_read_input_tokens,
  cache_creation_tokens: usage.cache_creation_input_tokens,
  output_tokens: usage.output_tokens,
  provider: getAPIProviderForStatsig(),
  has_tools: options.tools.length > 0,
  thinking_enabled: options.thinkingConfig.type === 'enabled',
})
```

## 关键文件

- [src/services/api/claude.ts](src/services/api/claude.ts) — API 客户端
- [src/services/api/withRetry.ts](src/services/api/withRetry.ts) — 重试逻辑
- [src/services/api/errors.ts](src/services/api/errors.ts) — 错误定义
- [src/utils/auth.ts](src/utils/auth.ts) — 认证管理

## 关联文档

- [09-API与网络/API重试机制-API-Retry-Mechanism.md](API重试机制-API-Retry-Mechanism.md)
- [09-API与网络/错误处理模式-Error-Handling-Patterns.md](错误处理模式-Error-Handling-Patterns.md)