# 调试工具

## 概述

Claude Code 提供了多种调试工具和日志机制，帮助开发者诊断问题和优化性能。

## 核心文件

- [src/utils/debug.ts](src/utils/debug.ts) — 调试工具
- [src/utils/log.ts](src/utils/log.ts) — 日志系统
- [src/services/diagnosticTracking.ts](src/services/diagnosticTracking.ts) — 诊断追踪

## 调试日志

### logForDebugging

```typescript
import { logForDebugging } from 'src/utils/debug.js'

// 基本调试日志
logForDebugging('Processing message')

// 带数据的日志
logForDebugging('API response', { tokens_used: 1234, duration_ms: 567 })

// 带级别的日志
logForDebugging('Retry attempt', { attempt: 3 }, { level: 'warn' })
```

### 日志级别

```typescript
type LogLevel = 'debug' | 'info' | 'warn' | 'error'

function logForDebugging(
  message: string,
  data?: Record<string, unknown>,
  options?: { level?: LogLevel },
): void {
  if (!isDebugEnabled() && options?.level !== 'error') {
    return
  }

  const timestamp = new Date().toISOString()
  const level = options?.level ?? 'debug'

  console.error(`[${timestamp}] [${level.toUpperCase()}] ${message}`, data ?? '')
}
```

### 启用调试

```bash
# 启用调试日志
export CLAUDE_DEBUG=1

# 或使用 --debug 标志
claude --debug
```

## 错误日志

### logError

```typescript
import { logError } from 'src/utils/log.js'

// 记录错误
try {
  await riskyOperation()
} catch (error) {
  logError(error)
}

// 带上下文的错误
logError(error, { operation: 'file_read', path: filePath })
```

### 错误追踪

```typescript
export function logError(
  error: unknown,
  context?: Record<string, unknown>,
): void {
  const message = errorMessage(error)
  const stack = error instanceof Error ? error.stack : undefined

  // 输出到 stderr
  console.error({
    timestamp: new Date().toISOString(),
    message,
    stack,
    context,
  })

  // 发送到遥测
  logEvent('tengu_error', {
    error_type: error?.constructor?.name ?? 'Unknown',
    error_message: message,
    has_stack: !!stack,
    ...context,
  })
}
```

## 诊断追踪

### 追踪 ID

```typescript
import {
  startDiagnosticTracking,
  endDiagnosticTracking,
} from 'src/services/diagnosticTracking.js'

// 开始追踪
const trackingId = startDiagnosticTracking('api_call', {
  model: 'opus',
  endpoint: '/messages',
})

try {
  const result = await apiCall()
  endDiagnosticTracking(trackingId, { success: true })
  return result
} catch (error) {
  endDiagnosticTracking(trackingId, { success: false, error: errorMessage(error) })
  throw error
}
```

### 性能追踪

```typescript
export function trackPerformance<T>(
  name: string,
  fn: () => T,
): T {
  const startTime = performance.now()
  const trackingId = startDiagnosticTracking(name)

  try {
    const result = fn()
    const duration = performance.now() - startTime
    endDiagnosticTracking(trackingId, { duration_ms: duration })
    return result
  } catch (error) {
    endDiagnosticTracking(trackingId, { error: true })
    throw error
  }
}

// 使用
const result = trackPerformance('process_message', () => {
  return processMessage(msg)
})
```

## 查询分析器

### Query Profiler

```typescript
import {
  queryCheckpoint,
  getQueryProfile,
} from 'src/utils/queryProfiler.js'

async function processQuery(input: string) {
  queryCheckpoint('query_start')

  const parsed = parseInput(input)
  queryCheckpoint('query_parse_done')

  const response = await apiCall(parsed)
  queryCheckpoint('query_api_done')

  const result = processResponse(response)
  queryCheckpoint('query_process_done')

  return result
}

// 获取性能分析
const profile = getQueryProfile()
console.log(profile)
// {
//   query_start: 0,
//   query_parse_done: 5,
//   query_api_done: 1234,
//   query_process_done: 1250,
// }
```

## 开发栏

### DevBar 组件

```typescript
export function DevBar(): React.ReactElement | null {
  if (!isDebugEnabled()) return null

  const appState = useAppState()

  return (
    <Box flexDirection="column" borderStyle="single">
      <Text dimColor>=== Dev Mode ===</Text>
      <Text dimColor>Messages: {appState.messages.length}</Text>
      <Text dimColor>Processing: {appState.isProcessing ? 'Yes' : 'No'}</Text>
      <Text dimColor>Tokens: {appState.totalTokensUsed}</Text>
    </Box>
  )
}
```

## 状态检查

### 检查点日志

```typescript
export function logCheckpoint(name: string, data?: unknown): void {
  if (!isDebugEnabled()) return

  console.error(`[CHECKPOINT] ${name}`, {
    timestamp: Date.now(),
    memory: process.memoryUsage(),
    ...data,
  })
}
```

### 内存监控

```typescript
export function logMemoryUsage(label: string): void {
  const usage = process.memoryUsage()
  logForDebugging(`Memory [${label}]`, {
    heap_used: formatBytes(usage.heapUsed),
    heap_total: formatBytes(usage.heapTotal),
    rss: formatBytes(usage.rss),
    external: formatBytes(usage.external),
  })
}

function formatBytes(bytes: number): string {
  return `${(bytes / 1024 / 1024).toFixed(2)} MB`
}
```

## 调试命令

### /debug 命令

```typescript
export const debugCommand: Command = {
  type: 'local',
  name: 'debug',
  description: 'Toggle debug mode',
  async action() {
    const current = isDebugEnabled()
    setDebugEnabled(!current)
    return `Debug mode: ${!current ? 'ON' : 'OFF'}`
  },
}
```

### /status 命令

```typescript
export const statusCommand: Command = {
  type: 'local',
  name: 'status',
  description: 'Show system status',
  async action(context) {
    const appState = context.getAppState()
    return JSON.stringify({
      messages: appState.messages.length,
      tokens: appState.totalTokensUsed,
      processing: appState.isProcessing,
      memory: process.memoryUsage(),
    }, null, 2)
  },
}
```

## 断点调试

### Node.js 调试

```bash
# 启动调试模式
node --inspect-brk dist/entrypoints/cli.js

# 或使用 Bun
bun --inspect-brk src/bootstrap-entry.ts
```

### 条件断点

```typescript
if (process.env.DEBUG_BREAK === 'true') {
  debugger  // 触发断点
}
```

## 日志文件

### 写入日志文件

```typescript
import { getClaudeTempDir } from 'src/utils/permissions/filesystem.js'

export async function writeDebugLog(
  filename: string,
  content: string,
): Promise<string> {
  const logDir = join(getClaudeTempDir(), 'logs')
  await mkdir(logDir, { recursive: true })

  const logPath = join(logDir, filename)
  await appendFile(logPath, content + '\n')

  return logPath
}
```

### 日志轮转

```typescript
export async function rotateLogs(
  logDir: string,
  maxFiles: number = 5,
): Promise<void> {
  const files = await readdir(logDir)
  const logFiles = files.filter(f => f.endsWith('.log')).sort().reverse()

  // 删除旧日志
  for (const file of logFiles.slice(maxFiles)) {
    await unlink(join(logDir, file))
  }
}
```

## 性能分析

### CPU 分析

```bash
# 启用 CPU 分析
node --cpu-prof dist/entrypoints/cli.js

# 输出文件：CPU-*.cpuprofile
# 可在 Chrome DevTools 中查看
```

### 堆快照

```bash
# 启用堆快照
node --heap-prof dist/entrypoints/cli.js

# 输出文件：Heap-*.heapprofile
```

## 调试工具函数

### 延迟执行

```typescript
export function debugDelay(ms: number): Promise<void> {
  if (!isDebugEnabled()) return Promise.resolve()
  return new Promise(resolve => setTimeout(resolve, ms))
}
```

### 随机失败（测试用）

```typescript
export function debugRandomFailure(
  probability: number = 0.1,
  message: string = 'Debug random failure',
): void {
  if (!isDebugEnabled()) return
  if (Math.random() < probability) {
    throw new Error(message)
  }
}
```

## 关键文件

- [src/utils/debug.ts](src/utils/debug.ts) — 调试工具
- [src/utils/log.ts](src/utils/log.ts) — 日志系统
- [src/utils/queryProfiler.ts](src/utils/queryProfiler.ts) — 性能分析
- [src/components/DevBar.tsx](src/components/DevBar.tsx) — 开发栏

## 关联文档

- [18-调试测试/测试策略-Testing-Strategy.md](测试策略-Testing-Strategy.md)
- [14-服务层/分析服务-Analytics-Service.md](../14-服务层/分析服务-Analytics-Service.md)