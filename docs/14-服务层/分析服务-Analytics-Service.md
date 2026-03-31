# 分析服务

## 概述

分析服务（Analytics Service）负责收集、处理和发送遥测数据，用于监控 Claude Code 的使用情况、性能指标和错误追踪。

## 核心文件

- [src/services/analytics/index.ts](src/services/analytics/index.ts) — 公共 API
- [src/services/analytics/sink.ts](src/services/analytics/sink.ts) — 数据接收器
- [src/services/analytics/datadog.ts](src/services/analytics/datadog.ts) — Datadog 集成
- [src/services/analytics/firstPartyEventLogger.ts](src/services/analytics/firstPartyEventLogger.ts) — 一方日志

## 架构

```
logEvent() 调用
    ↓
事件队列（启动前缓冲）
    ↓
AnalyticsSink（启动后附加）
    ├── Datadog 后端
    ├── 一方事件日志
    └── 采样过滤
    ↓
远程服务
```

## 公共 API

### logEvent

```typescript
/**
 * 记录事件到分析后端（同步）
 * 如果 Sink 未附加，事件会排队等待
 */
export function logEvent(
  eventName: string,
  metadata: LogEventMetadata,
): void {
  if (sink === null) {
    eventQueue.push({ eventName, metadata, async: false })
    return
  }
  sink.logEvent(eventName, metadata)
}

/**
 * 记录事件到分析后端（异步）
 */
export async function logEventAsync(
  eventName: string,
  metadata: LogEventMetadata,
): Promise<void> {
  if (sink === null) {
    eventQueue.push({ eventName, metadata, async: true })
    return
  }
  await sink.logEventAsync(eventName, metadata)
}
```

### 元数据类型

```typescript
// 基本元数据类型
type LogEventMetadata = {
  [key: string]: boolean | number | undefined
}

// 字符串值需要类型标记确认安全
export type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS = never

// 使用示例
logEvent('tengu_api_success', {
  model: 'opus' as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS,
  tokens_used: 1234,
  duration_ms: 567,
})
```

### PII 标记类型

```typescript
// PII 标记类型（用于敏感数据）
export type AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED = never

// PII 字段会被过滤，只发送到一方后端
logEvent('tengu_user_action', {
  _PROTO_user_email: email as AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED,
  action: 'login',
})

// 过滤 PII 字段
export function stripProtoFields<V>(
  metadata: Record<string, V>,
): Record<string, V> {
  let result: Record<string, V> | undefined
  for (const key in metadata) {
    if (key.startsWith('_PROTO_')) {
      if (result === undefined) {
        result = { ...metadata }
      }
      delete result[key]
    }
  }
  return result ?? metadata
}
```

## Sink 接口

### AnalyticsSink

```typescript
export type AnalyticsSink = {
  logEvent: (eventName: string, metadata: LogEventMetadata) => void
  logEventAsync: (
    eventName: string,
    metadata: LogEventMetadata,
  ) => Promise<void>
}
```

### 附加 Sink

```typescript
let sink: AnalyticsSink | null = null
const eventQueue: QueuedEvent[] = []

export function attachAnalyticsSink(newSink: AnalyticsSink): void {
  // 幂等：已附加则跳过
  if (sink !== null) {
    return
  }
  sink = newSink

  // 异步排空队列（不阻塞启动）
  if (eventQueue.length > 0) {
    const queuedEvents = [...eventQueue]
    eventQueue.length = 0

    // 记录队列大小（用于调试）
    if (process.env.USER_TYPE === 'ant') {
      sink.logEvent('analytics_sink_attached', {
        queued_event_count: queuedEvents.length,
      })
    }

    queueMicrotask(() => {
      for (const event of queuedEvents) {
        if (event.async) {
          void sink!.logEventAsync(event.eventName, event.metadata)
        } else {
          sink!.logEvent(event.eventName, event.metadata)
        }
      }
    })
  }
}
```

## Datadog 集成

### 事件发送

```typescript
export function createDatadogSink(): AnalyticsSink {
  return {
    logEvent(eventName, metadata) {
      const event = {
        eventName,
        timestamp: Date.now(),
        metadata: stripProtoFields(metadata),
        // 添加通用标签
        tags: {
          version: VERSION,
          platform: process.platform,
          user_type: process.env.USER_TYPE,
        },
      }

      // 发送到 Datadog
      sendToDatadog(event)
    },

    async logEventAsync(eventName, metadata) {
      // 异步版本
      await sendToDatadogAsync({
        eventName,
        timestamp: Date.now(),
        metadata: stripProtoFields(metadata),
      })
    },
  }
}
```

## 一方事件日志

### First Party Logger

```typescript
export class FirstPartyEventLogger {
  private queue: Event[] = []
  private flushInterval: Timer

  constructor(private config: FirstPartyConfig) {
    // 定期刷新
    this.flushInterval = setInterval(
      () => this.flush(),
      config.flushIntervalMs,
    )
  }

  logEvent(eventName: string, metadata: LogEventMetadata): void {
    this.queue.push({
      event_name: eventName,
      timestamp: new Date().toISOString(),
      ...metadata,
    })

    // 队列满时立即刷新
    if (this.queue.length >= this.config.batchSize) {
      void this.flush()
    }
  }

  private async flush(): Promise<void> {
    if (this.queue.length === 0) return

    const events = [...this.queue]
    this.queue = []

    try {
      await this.sendBatch(events)
    } catch (error) {
      // 失败时重新入队
      this.queue.unshift(...events)
      logError(error)
    }
  }

  private async sendBatch(events: Event[]): Promise<void> {
    await fetch(this.config.endpoint, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${this.config.apiKey}`,
      },
      body: JSON.stringify({ events }),
    })
  }
}
```

## 采样配置

### 事件采样

```typescript
interface SamplingConfig {
  events: Record<string, {
    sample_rate: number  // 0.0 - 1.0
  }>
}

function shouldSample(
  eventName: string,
  config: SamplingConfig,
): boolean {
  const eventConfig = config.events[eventName]
  if (!eventConfig) return true

  return Math.random() < eventConfig.sample_rate
}

function applySampling(
  eventName: string,
  metadata: LogEventMetadata,
  config: SamplingConfig,
): LogEventMetadata | null {
  const eventConfig = config.events[eventName]
  if (!eventConfig) return metadata

  if (Math.random() >= eventConfig.sample_rate) {
    return null  // 丢弃
  }

  return {
    ...metadata,
    sample_rate: eventConfig.sample_rate,
  }
}
```

## Kill Switch

### 紧急关闭

```typescript
// sinkKillswitch.ts
let killswitchActive = false

export function isKillswitchActive(): boolean {
  return killswitchActive
}

export function setKillswitch(active: boolean): void {
  killswitchActive = active
}

// 在 Sink 中检查
function logEvent(eventName: string, metadata: LogEventMetadata): void {
  if (isKillswitchActive()) {
    return  // 紧急关闭，丢弃所有事件
  }

  // 正常处理
  ...
}
```

## 元数据丰富

### 通用元数据

```typescript
export function enrichMetadata(
  metadata: LogEventMetadata,
): EnrichedMetadata {
  return {
    ...metadata,
    // 会话信息
    session_id: getSessionId(),
    client_id: getClientId(),

    // 环境信息
    version: VERSION,
    platform: process.platform,
    arch: process.arch,

    // 用户类型
    user_type: process.env.USER_TYPE,
    is_subscriber: isClaudeAISubscriber(),

    // 时间戳
    timestamp: Date.now(),
  }
}
```

## GrowthBook 集成

### 特性开关

```typescript
// growthbook.ts
import { GrowthBook } from '@growthbook/growthbook'

const growthbook = new GrowthBook({
  apiHost: 'https://cdn.growthbook.io',
  clientKey: process.env.GROWTHBOOK_CLIENT_KEY,
})

export async function initializeGrowthBook(): Promise<void> {
  await growthbook.loadFeatures()
}

export function isFeatureEnabled(featureKey: string): boolean {
  return growthbook.isOn(featureKey)
}

export function getFeatureValue<T>(featureKey: string, defaultValue: T): T {
  return growthbook.getFeatureValue(featureKey, defaultValue)
}
```

## 调试支持

### 开发模式日志

```typescript
if (process.env.DEBUG_ANALYTICS) {
  console.log('[Analytics]', eventName, JSON.stringify(metadata, null, 2))
}
```

### 测试重置

```typescript
export function _resetForTesting(): void {
  sink = null
  eventQueue.length = 0
}
```

## 常见事件

### API 事件

```typescript
logEvent('tengu_api_success', {
  model,
  tokens_used,
  duration_ms,
  cache_read_tokens,
})

logEvent('tengu_api_error', {
  model,
  error_type: classifyAPIError(error),
  status_code: error.status,
})
```

### 工具事件

```typescript
logEvent('tengu_tool_use', {
  tool_name: tool.name,
  duration_ms,
  success: !result.is_error,
})
```

### 会话事件

```typescript
logEvent('tengu_session_start', {
  session_type: 'cli',
})

logEvent('tengu_session_end', {
  duration_ms,
  message_count,
  tokens_used,
})
```

## 关键文件

- [src/services/analytics/index.ts](src/services/analytics/index.ts) — 公共 API
- [src/services/analytics/sink.ts](src/services/analytics/sink.ts) — Sink 实现
- [src/services/analytics/datadog.ts](src/services/analytics/datadog.ts) — Datadog
- [src/services/analytics/firstPartyEventLogger.ts](src/services/analytics/firstPartyEventLogger.ts) — 一方日志
- [src/services/analytics/metadata.ts](src/services/analytics/metadata.ts) — 元数据丰富

## 关联文档

- [14-服务层/服务层详解-Services-Detail.md](服务层详解-Services-Detail.md)
- [18-调试测试/调试工具-Debugging-Tools.md](../18-调试测试/调试工具-Debugging-Tools.md)