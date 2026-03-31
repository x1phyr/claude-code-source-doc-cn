# 错误处理模式

## 概述

Claude Code 实现了完整的错误处理体系，包括自定义错误类型、API 错误处理、错误恢复和用户友好的错误消息。

## 核心文件

- [src/utils/errors.ts](src/utils/errors.ts) — 自定义错误类型
- [src/services/api/errors.ts](src/services/api/errors.ts) — API 错误处理
- [src/services/api/withRetry.ts](src/services/api/withRetry.ts) — 重试机制

## 错误类型

### 自定义错误类

```typescript
// 基础错误类
export class ClaudeError extends Error {
  constructor(message: string) {
    super(message)
    this.name = this.constructor.name
  }
}

// 命令格式错误
export class MalformedCommandError extends Error {}

// 中止错误
export class AbortError extends Error {
  constructor(message?: string) {
    super(message)
    this.name = 'AbortError'
  }
}

// Shell 命令错误
export class ShellError extends Error {
  constructor(
    public readonly stdout: string,
    public readonly stderr: string,
    public readonly code: number,
    public readonly interrupted: boolean,
  ) {
    super('Shell command failed')
    this.name = 'ShellError'
  }
}

// 配置解析错误
export class ConfigParseError extends Error {
  filePath: string
  defaultConfig: unknown

  constructor(message: string, filePath: string, defaultConfig: unknown) {
    super(message)
    this.name = 'ConfigParseError'
    this.filePath = filePath
    this.defaultConfig = defaultConfig
  }
}

// 远程操作错误
export class TeleportOperationError extends Error {
  constructor(
    message: string,
    public readonly formattedMessage: string,
  ) {
    super(message)
    this.name = 'TeleportOperationError'
  }
}
```

### 遥测安全错误

```typescript
/**
 * 消息安全可记录到遥测的错误。
 * 使用长名称确认消息不含敏感数据（文件路径、URL、代码片段）。
 */
export class TelemetrySafeError_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS extends Error {
  readonly telemetryMessage: string

  constructor(message: string, telemetryMessage?: string) {
    super(message)
    this.name = 'TelemetrySafeError'
    this.telemetryMessage = telemetryMessage ?? message
  }
}

// 使用示例
throw new TelemetrySafeError_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS(
  `MCP tool timed out after ${ms}ms`,  // 完整消息
  'MCP tool timed out'                  // 遥测消息
)
```

## 错误类型判断

### 中止错误检测

```typescript
export function isAbortError(e: unknown): boolean {
  return (
    e instanceof AbortError ||
    e instanceof APIUserAbortError ||
    (e instanceof Error && e.name === 'AbortError')
  )
}
```

### 文件系统错误

```typescript
// 获取 errno 代码
export function getErrnoCode(e: unknown): string | undefined {
  if (e && typeof e === 'object' && 'code' in e && typeof e.code === 'string') {
    return e.code
  }
  return undefined
}

// 检测 ENOENT
export function isENOENT(e: unknown): boolean {
  return getErrnoCode(e) === 'ENOENT'
}

// 检测文件不可访问
export function isFsInaccessible(e: unknown): e is NodeJS.ErrnoException {
  const code = getErrnoCode(e)
  return (
    code === 'ENOENT' ||   // 不存在
    code === 'EACCES' ||   // 权限拒绝
    code === 'EPERM' ||    // 操作不允许
    code === 'ENOTDIR' ||  // 不是目录
    code === 'ELOOP'       // 符号链接循环
  )
}
```

### Axios 错误分类

```typescript
export type AxiosErrorKind =
  | 'auth'      // 401/403
  | 'timeout'   // ECONNABORTED
  | 'network'   // ECONNREFUSED/ENOTFOUND
  | 'http'      // 其他 HTTP 错误
  | 'other'     // 非 Axios 错误

export function classifyAxiosError(e: unknown): {
  kind: AxiosErrorKind
  status?: number
  message: string
} {
  const message = errorMessage(e)
  if (!e || typeof e !== 'object' || !('isAxiosError' in e)) {
    return { kind: 'other', message }
  }

  const err = e as { response?: { status?: number }, code?: string }
  const status = err.response?.status

  if (status === 401 || status === 403) {
    return { kind: 'auth', status, message }
  }
  if (err.code === 'ECONNABORTED') {
    return { kind: 'timeout', status, message }
  }
  if (err.code === 'ECONNREFUSED' || err.code === 'ENOTFOUND') {
    return { kind: 'network', status, message }
  }
  return { kind: 'http', status, message }
}
```

## API 错误处理

### 错误消息常量

```typescript
export const API_ERROR_MESSAGE_PREFIX = 'API Error'
export const PROMPT_TOO_LONG_ERROR_MESSAGE = 'Prompt is too long'
export const CREDIT_BALANCE_TOO_LOW_ERROR_MESSAGE = 'Credit balance is too low'
export const INVALID_API_KEY_ERROR_MESSAGE = 'Not logged in · Please run /login'
export const TOKEN_REVOKED_ERROR_MESSAGE = 'OAuth token revoked · Please run /login'
export const API_TIMEOUT_ERROR_MESSAGE = 'Request timed out'
export const REPEATED_529_ERROR_MESSAGE = 'Repeated 529 Overloaded errors'
```

### 创建错误消息

```typescript
export function getAssistantMessageFromError(
  error: unknown,
  model: string,
  options?: { messages?: Message[], messagesForAPI?: (UserMessage | AssistantMessage)[] },
): AssistantMessage {
  // 超时错误
  if (error instanceof APIConnectionTimeoutError) {
    return createAssistantAPIErrorMessage({
      content: API_TIMEOUT_ERROR_MESSAGE,
      error: 'unknown',
    })
  }

  // 图片大小错误
  if (error instanceof ImageSizeError || error instanceof ImageResizeError) {
    return createAssistantAPIErrorMessage({
      content: getImageTooLargeErrorMessage(),
    })
  }

  // Prompt 太长错误
  if (error instanceof Error && error.message.toLowerCase().includes('prompt is too long')) {
    return createAssistantAPIErrorMessage({
      content: PROMPT_TOO_LONG_ERROR_MESSAGE,
      error: 'invalid_request',
      errorDetails: error.message,
    })
  }

  // PDF 错误
  if (error instanceof Error && /maximum of \d+ PDF pages/.test(error.message)) {
    return createAssistantAPIErrorMessage({
      content: getPdfTooLargeErrorMessage(),
      error: 'invalid_request',
      errorDetails: error.message,
    })
  }

  // 认证错误
  if (error instanceof Error && error.message.toLowerCase().includes('x-api-key')) {
    return createAssistantAPIErrorMessage({
      error: 'authentication_failed',
      content: INVALID_API_KEY_ERROR_MESSAGE,
    })
  }

  // OAuth Token 撤销
  if (error instanceof APIError && error.status === 403 &&
      error.message.includes('OAuth token has been revoked')) {
    return createAssistantAPIErrorMessage({
      error: 'authentication_failed',
      content: getTokenRevokedErrorMessage(),
    })
  }

  // 组织禁用
  if (error instanceof APIError && error.status === 400 &&
      error.message.toLowerCase().includes('organization has been disabled')) {
    return createAssistantAPIErrorMessage({
      error: 'invalid_request',
      content: ORG_DISABLED_ERROR_MESSAGE,
    })
  }

  // 通用错误
  if (error instanceof Error) {
    return createAssistantAPIErrorMessage({
      content: `${API_ERROR_MESSAGE_PREFIX}: ${error.message}`,
      error: 'unknown',
    })
  }

  return createAssistantAPIErrorMessage({
    content: API_ERROR_MESSAGE_PREFIX,
    error: 'unknown',
  })
}
```

### API 错误分类

```typescript
export function classifyAPIError(error: unknown): string {
  // 中止请求
  if (error instanceof Error && error.message === 'Request was aborted.') {
    return 'aborted'
  }

  // 超时
  if (error instanceof APIConnectionTimeoutError) {
    return 'api_timeout'
  }

  // 重复 529
  if (error instanceof Error && error.message.includes(REPEATED_529_ERROR_MESSAGE)) {
    return 'repeated_529'
  }

  // 容量关闭
  if (error instanceof Error && error.message.includes(CUSTOM_OFF_SWITCH_MESSAGE)) {
    return 'capacity_off_switch'
  }

  // 速率限制
  if (error instanceof APIError && error.status === 429) {
    return 'rate_limit'
  }

  // 服务器过载
  if (error instanceof APIError && error.status === 529) {
    return 'server_overload'
  }

  // Prompt 太长
  if (error instanceof Error && error.message.toLowerCase().includes('prompt is too long')) {
    return 'prompt_too_long'
  }

  // PDF 错误
  if (error instanceof Error && /maximum of \d+ PDF pages/.test(error.message)) {
    return 'pdf_too_large'
  }

  // 认证错误
  if (error instanceof APIError && (error.status === 401 || error.status === 403)) {
    if (error.message.includes('OAuth token has been revoked')) {
      return 'token_revoked'
    }
    return 'auth_error'
  }

  // 服务器错误
  if (error instanceof APIError && error.status >= 500) {
    return 'server_error'
  }

  // 客户端错误
  if (error instanceof APIError && error.status >= 400) {
    return 'client_error'
  }

  // 连接错误
  if (error instanceof APIConnectionError) {
    return 'connection_error'
  }

  return 'unknown'
}
```

## 错误恢复

### 短堆栈跟踪

```typescript
/**
 * 提取错误消息 + 前 N 个堆栈帧
 * 用于工具结果中发送给模型，避免浪费 Token
 */
export function shortErrorStack(e: unknown, maxFrames = 5): string {
  if (!(e instanceof Error)) return String(e)
  if (!e.stack) return e.message

  const lines = e.stack.split('\n')
  const header = lines[0] ?? e.message
  const frames = lines.slice(1).filter(l => l.trim().startsWith('at '))

  if (frames.length <= maxFrames) return e.stack

  return [header, ...frames.slice(0, maxFrames)].join('\n')
}
```

### 错误消息提取

```typescript
// 提取字符串消息
export function errorMessage(e: unknown): string {
  return e instanceof Error ? e.message : String(e)
}

// 转换为 Error 对象
export function toError(e: unknown): Error {
  return e instanceof Error ? e : new Error(String(e))
}

// 检查特定错误消息
export function hasExactErrorMessage(error: unknown, message: string): boolean {
  return error instanceof Error && error.message === message
}
```

## 用户友好错误

### 交互模式感知

```typescript
// 根据交互模式生成不同消息
export function getPdfTooLargeErrorMessage(): string {
  const limits = `max ${API_PDF_MAX_PAGES} pages, ${formatFileSize(PDF_TARGET_RAW_SIZE)}`
  return getIsNonInteractiveSession()
    ? `PDF too large (${limits}). Try reading the file a different way.`
    : `PDF too large (${limits}). Double press esc to go back and try again.`
}

export function getTokenRevokedErrorMessage(): string {
  return getIsNonInteractiveSession()
    ? 'Your account does not have access to Claude. Please login again.'
    : TOKEN_REVOKED_ERROR_MESSAGE
}
```

### CCR 模式特殊处理

```typescript
// CCR (Claude Code Remote) 模式使用 JWT 认证
function isCCRMode(): boolean {
  return isEnvTruthy(process.env.CLAUDE_CODE_REMOTE)
}

// CCR 模式下认证错误建议重试而非登录
if (error instanceof Error && error.message.toLowerCase().includes('x-api-key')) {
  if (isCCRMode()) {
    return createAssistantAPIErrorMessage({
      error: 'authentication_failed',
      content: CCR_AUTH_ERROR_MESSAGE,  // 建议重试
    })
  }
  return createAssistantAPIErrorMessage({
    error: 'authentication_failed',
    content: INVALID_API_KEY_ERROR_MESSAGE,  // 建议登录
  })
}
```

## 错误日志

### 统计事件

```typescript
logEvent('tengu_tool_use_tool_result_mismatch_error', {
  toolUseId,
  normalizedSequence,
  preNormalizedSequence,
  normalizedMessageCount,
  originalMessageCount,
})

logEvent('tengu_unexpected_tool_result', {})

logEvent('tengu_duplicate_tool_use_id', {})

logEvent('tengu_refusal_api_response', {})
```

### 错误上下文

```typescript
// 记录错误上下文
function logErrorWithContext(
  error: unknown,
  context: {
    operation: string
    filePath?: string
    toolName?: string
  },
): void {
  const message = errorMessage(error)
  const classification = classifyAPIError(error)

  logError(error)
  logEvent('tengu_error', {
    classification,
    operation: context.operation,
    file_path: context.filePath,
    tool_name: context.toolName,
    message_length: message.length,
  })
}
```

## 工具错误结果

### Tool Result 错误

```typescript
// 工具执行错误处理
export function createToolErrorResult(
  error: unknown,
  toolName: string,
): ToolResult {
  const message = errorMessage(error)

  // 特定错误类型处理
  if (isAbortError(error)) {
    return {
      output: 'Operation was cancelled',
      is_error: true,
    }
  }

  if (isFsInaccessible(error)) {
    return {
      output: `File or directory not accessible: ${message}`,
      is_error: true,
    }
  }

  // 通用错误
  return {
    output: `${toolName} failed: ${shortErrorStack(error, 3)}`,
    is_error: true,
  }
}
```

## 错误边界

### React 错误边界

```typescript
export class SentryErrorBoundary extends React.Component<{
  children: React.ReactNode
  fallback?: React.ReactNode
}> {
  state = { hasError: false, error: null }

  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error }
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    logError(error)
    logEvent('tengu_react_error_boundary', {
      component_stack: errorInfo.componentStack,
      error_message: error.message,
    })
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback ?? (
        <Text color="red">An error occurred. Please restart Claude Code.</Text>
      )
    }
    return this.props.children
  }
}
```

## 统计日志

```typescript
logEvent('tengu_api_error', {
  classification: classifyAPIError(error),
  status: error.status,
  model,
  attempt: retryCount,
})

logEvent('tengu_shell_error', {
  command: commandName,
  exit_code: code,
  stderr_length: stderr.length,
})
```

## 关键文件

- [src/utils/errors.ts](src/utils/errors.ts) — 错误类型
- [src/services/api/errors.ts](src/services/api/errors.ts) — API 错误
- [src/services/api/errorUtils.ts](src/services/api/errorUtils.ts) — 错误工具
- [src/components/SentryErrorBoundary.ts](src/components/SentryErrorBoundary.ts) — 错误边界

## 关联文档

- [09-API与网络/API重试机制-API-Retry-Mechanism.md](API重试机制-API-Retry-Mechanism.md)
- [09-API与网络/API服务详解-API-Service.md](API服务详解-API-Service.md)
- [03-工具系统/工具系统详解-Tool-System.md](../03-工具系统/工具系统详解-Tool-System.md)