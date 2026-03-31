# API 重试机制

## 概述

API 重试机制是 Claude Code 与 Anthropic API 通信的核心容错层，处理速率限制、服务过载、网络错误等场景，确保请求可靠完成。

## 核心文件

- [src/services/api/withRetry.ts](src/services/api/withRetry.ts) — 重试逻辑
- [src/services/api/errors.ts](src/services/api/errors.ts) — 错误处理
- [src/services/api/claude.ts](src/services/api/claude.ts) — API 客户端

## 重试策略

### 指数退避

```typescript
export function getRetryDelay(
  attempt: number,
  retryAfterHeader?: string | null,
  maxDelayMs = 32000,
): number {
  // 优先使用 Retry-After 头
  if (retryAfterHeader) {
    const seconds = parseInt(retryAfterHeader, 10)
    if (!isNaN(seconds)) {
      return seconds * 1000
    }
  }

  // 指数退避 + 抖动
  const baseDelay = Math.min(
    BASE_DELAY_MS * Math.pow(2, attempt - 1),  // 500 * 2^(n-1)
    maxDelayMs,  // 最大 32 秒
  )
  const jitter = Math.random() * 0.25 * baseDelay  // 0-25% 抖动
  return baseDelay + jitter
}
```

### 延迟序列示例

| 尝试 | 基础延迟 | 加抖动后 |
|------|----------|----------|
| 1 | 500ms | 500-625ms |
| 2 | 1000ms | 1000-1250ms |
| 3 | 2000ms | 2000-2500ms |
| 4 | 4000ms | 4000-5000ms |
| 5 | 8000ms | 8000-10000ms |
| 6 | 16000ms | 16000-20000ms |
| 7+ | 32000ms | 32000-40000ms |

## 重试入口

### withRetry 函数

```typescript
export async function* withRetry<T>(
  getClient: () => Promise<Anthropic>,
  operation: (
    client: Anthropic,
    attempt: number,
    context: RetryContext,
  ) => Promise<T>,
  options: RetryOptions,
): AsyncGenerator<SystemAPIErrorMessage, T>
```

### 重试选项

```typescript
interface RetryOptions {
  maxRetries?: number           // 最大重试次数，默认 10
  model: string                 // 模型名称
  fallbackModel?: string        // 回退模型
  thinkingConfig: ThinkingConfig
  fastMode?: boolean            // 快速模式
  signal?: AbortSignal          // 中止信号
  querySource?: QuerySource     // 查询来源
  initialConsecutive529Errors?: number  // 初始 529 计数
}
```

## 错误类型处理

### 529 过载错误

```typescript
function is529Error(error: unknown): boolean {
  if (!(error instanceof APIError)) {
    return false
  }
  return (
    error.status === 529 ||
    (error.message?.includes('"type":"overloaded_error"') ?? false)
  )
}
```

**前台源重试**：

```typescript
const FOREGROUND_529_RETRY_SOURCES = new Set<QuerySource>([
  'repl_main_thread',
  'sdk',
  'agent:custom',
  'agent:default',
  'agent:builtin',
  'compact',
  'hook_agent',
  'hook_prompt',
  'verification_agent',
  'side_question',
  'auto_mode',
  'bash_classifier',
])

function shouldRetry529(querySource: QuerySource | undefined): boolean {
  return querySource === undefined || FOREGROUND_529_RETRY_SOURCES.has(querySource)
}
```

**后台源立即失败**：非前台源（摘要、标题、建议、分类器）在 529 时立即失败，避免容量级联期间的重试放大。

### 429 速率限制

```typescript
if (error.status === 429) {
  return !isClaudeAISubscriber() || isEnterpriseSubscriber()
}
```

### 网络错误

```typescript
function isStaleConnectionError(error: unknown): boolean {
  if (!(error instanceof APIConnectionError)) {
    return false
  }
  const details = extractConnectionErrorDetails(error)
  return details?.code === 'ECONNRESET' || details?.code === 'EPIPE'
}

// 禁用 keep-alive 并重连
if (isStaleConnection) {
  disableKeepAlive()
}
```

### 认证错误

```typescript
// 401 错误：清除缓存并重试
if (error.status === 401) {
  clearApiKeyHelperCache()
  return true
}

// OAuth token 撤销
if (isOAuthTokenRevokedError(error)) {
  await handleOAuth401Error(failedAccessToken)
  return true
}
```

### 云服务认证

```typescript
// AWS Bedrock
function isBedrockAuthError(error: unknown): boolean {
  return isAwsCredentialsProviderError(error) ||
    (error instanceof APIError && error.status === 403)
}

// GCP Vertex
function isVertexAuthError(error: unknown): boolean {
  return isGoogleAuthLibraryCredentialError(error) ||
    (error instanceof APIError && error.status === 401)
}
```

## 快速模式处理

### 短延迟重试

```typescript
const SHORT_RETRY_THRESHOLD_MS = 20 * 1000  // 20 秒

if (wasFastModeActive && error.status === 429 || is529Error(error)) {
  const retryAfterMs = getRetryAfterMs(error)
  if (retryAfterMs !== null && retryAfterMs < SHORT_RETRY_THRESHOLD_MS) {
    // 短等待：保持快速模式，保留 prompt cache
    await sleep(retryAfterMs, options.signal)
    continue
  }
  // 长等待：进入冷却，切换到标准速度
  triggerFastModeCooldown(Date.now() + cooldownMs, cooldownReason)
  retryContext.fastMode = false
}
```

### 冷却机制

```typescript
const DEFAULT_FAST_MODE_FALLBACK_HOLD_MS = 30 * 60 * 1000  // 30 分钟
const MIN_COOLDOWN_MS = 10 * 60 * 1000  // 10 分钟

// 触发冷却
triggerFastModeCooldown(Date.now() + cooldownMs, cooldownReason)

// 检查冷却状态
const wasFastModeActive = isFastModeEnabled()
  ? retryContext.fastMode && !isFastModeCooldown()
  : false
```

## 模型回退

### Opus 过载回退

```typescript
const MAX_529_RETRIES = 3

if (is529Error(error) && isNonCustomOpusModel(options.model)) {
  consecutive529Errors++
  if (consecutive529Errors >= MAX_529_RETRIES) {
    if (options.fallbackModel) {
      logEvent('tengu_api_opus_fallback_triggered', {
        original_model: options.model,
        fallback_model: options.fallbackModel,
      })
      throw new FallbackTriggeredError(options.model, options.fallbackModel)
    }
    throw new CannotRetryError(new Error(REPEATED_529_ERROR_MESSAGE), retryContext)
  }
}
```

## 上下文溢出处理

```typescript
// 解析 max_tokens 超限错误
export function parseMaxTokensContextOverflowError(error: APIError) {
  // 格式: "input length and `max_tokens` exceed context limit: 188059 + 20000 > 200000"
  const regex = /input length and `max_tokens` exceed context limit: (\d+) \+ (\d+) > (\d+)/
  const match = error.message.match(regex)
  if (match) {
    return {
      inputTokens: parseInt(match[1], 10),
      maxTokens: parseInt(match[2], 10),
      contextLimit: parseInt(match[3], 10),
    }
  }
}

// 调整 max_tokens 并重试
if (overflowData) {
  const safetyBuffer = 1000
  const availableContext = contextLimit - inputTokens - safetyBuffer
  const adjustedMaxTokens = Math.max(FLOOR_OUTPUT_TOKENS, availableContext, minRequired)
  retryContext.maxTokensOverride = adjustedMaxTokens
  continue
}
```

## 持久重试模式

### 非交互会话

```typescript
const PERSISTENT_MAX_BACKOFF_MS = 5 * 60 * 1000  // 5 分钟
const PERSISTENT_RESET_CAP_MS = 6 * 60 * 60 * 1000  // 6 小时
const HEARTBEAT_INTERVAL_MS = 30_000  // 30 秒

function isPersistentRetryEnabled(): boolean {
  return isEnvTruthy(process.env.CLAUDE_CODE_UNATTENDED_RETRY)
}

// 持久模式下的长时间等待
if (persistent && error.status === 429) {
  const resetDelay = getRateLimitResetDelayMs(error)
  delayMs = resetDelay ?? Math.min(
    getRetryDelay(persistentAttempt, retryAfter, PERSISTENT_MAX_BACKOFF_MS),
    PERSISTENT_RESET_CAP_MS,
  )
}

// 分块等待，定期发送心跳
while (remaining > 0) {
  yield createSystemAPIErrorMessage(error, remaining, reportedAttempt, maxRetries)
  const chunk = Math.min(remaining, HEARTBEAT_INTERVAL_MS)
  await sleep(chunk, options.signal)
  remaining -= chunk
}
```

## 错误类型

### CannotRetryError

```typescript
export class CannotRetryError extends Error {
  constructor(
    public readonly originalError: unknown,
    public readonly retryContext: RetryContext,
  ) {
    super(errorMessage(originalError))
    this.name = 'RetryError'
  }
}
```

### FallbackTriggeredError

```typescript
export class FallbackTriggeredError extends Error {
  constructor(
    public readonly originalModel: string,
    public readonly fallbackModel: string,
  ) {
    super(`Model fallback triggered: ${originalModel} -> ${fallbackModel}`)
    this.name = 'FallbackTriggeredError'
  }
}
```

## 重试判断

```typescript
function shouldRetry(error: APIError): boolean {
  // 模拟错误不重试
  if (isMockRateLimitError(error)) return false

  // 持久模式：429/529 始终重试
  if (isPersistentRetryEnabled() && isTransientCapacityError(error)) {
    return true
  }

  // CCR 模式：401/403 重试
  if (isEnvTruthy(process.env.CLAUDE_CODE_REMOTE) && (error.status === 401 || error.status === 403)) {
    return true
  }

  // 过载错误
  if (error.message?.includes('"type":"overloaded_error"')) {
    return true
  }

  // 上下文溢出
  if (parseMaxTokensContextOverflowError(error)) {
    return true
  }

  // x-should-retry 头
  const shouldRetryHeader = error.headers?.get('x-should-retry')
  if (shouldRetryHeader === 'true' && (!isClaudeAISubscriber() || isEnterpriseSubscriber())) {
    return true
  }
  if (shouldRetryHeader === 'false' && !(process.env.USER_TYPE === 'ant' && error.status >= 500)) {
    return false
  }

  // 网络错误
  if (error instanceof APIConnectionError) return true

  // HTTP 状态码
  if (error.status === 408) return true  // 请求超时
  if (error.status === 409) return true  // 锁超时
  if (error.status === 429) return !isClaudeAISubscriber() || isEnterpriseSubscriber()
  if (error.status === 401) { clearApiKeyHelperCache(); return true }
  if (error.status >= 500) return true

  return false
}
```

## 统计事件

```typescript
logEvent('tengu_api_retry', {
  attempt: reportedAttempt,
  delayMs: delayMs,
  error: error.message,
  status: error.status,
  provider: getAPIProviderForStatsig(),
})
```

## 关键文件

- [src/services/api/withRetry.ts](src/services/api/withRetry.ts) — 重试逻辑
- [src/services/api/errors.ts](src/services/api/errors.ts) — 错误定义
- [src/services/api/errorUtils.ts](src/services/api/errorUtils.ts) — 错误工具
- [src/utils/fastMode.ts](src/utils/fastMode.ts) — 快速模式

## 关联文档

- [09-API与网络/API服务详解-API-Service.md](API服务详解-API-Service.md)
- [09-API与网络/错误处理模式-Error-Handling-Patterns.md](错误处理模式-Error-Handling-Patterns.md)