# 远程会话管理

## 概述

远程会话管理负责处理 Claude Code 与远程服务（claude.ai、IDE 扩展）之间的会话连接、状态同步和消息传递。

## 核心文件

- [src/remote/RemoteSessionManager.ts](src/remote/RemoteSessionManager.ts) — 会话管理
- [src/remote/SessionsWebSocket.ts](src/remote/SessionsWebSocket.ts) — WebSocket 连接
- [src/remote/sdkMessageAdapter.ts](src/remote/sdkMessageAdapter.ts) — 消息适配

## 会话生命周期

### 创建会话

```typescript
export async function createRemoteSession(options: {
  sessionId?: string
  name?: string
  cwd?: string
}): Promise<RemoteSession> {
  // 生成或使用指定 ID
  const sessionId = options.sessionId ?? generateSessionId()

  // 创建会话目录
  const sessionDir = await createSessionDirectory(sessionId)

  // 初始化会话状态
  const session: RemoteSession = {
    id: sessionId,
    name: options.name,
    cwd: options.cwd ?? process.cwd(),
    status: 'initialized',
    createdAt: Date.now(),
    messages: [],
    tools: [],
  }

  // 持久化会话元数据
  await saveSessionMetadata(session)

  return session
}
```

### 会话状态

```typescript
export type SessionStatus =
  | 'initialized'   // 已初始化
  | 'connecting'    // 正在连接
  | 'connected'     // 已连接
  | 'active'        // 活跃中
  | 'idle'          // 空闲
  | 'compacting'    // 压缩中
  | 'error'         // 错误
  | 'terminated'    // 已终止

export interface RemoteSession {
  id: string
  name?: string
  cwd: string
  status: SessionStatus
  createdAt: number
  messages: Message[]
  tools: Tool[]
  abortController?: AbortController
}
```

## WebSocket 连接

### 连接管理

```typescript
export class SessionsWebSocket {
  private ws: WebSocket | null = null
  private reconnectAttempts = 0
  private maxReconnectAttempts = 5

  async connect(url: string, token: string): Promise<void> {
    return new Promise((resolve, reject) => {
      this.ws = new WebSocket(url, {
        headers: {
          Authorization: `Bearer ${token}`,
        },
      })

      this.ws.on('open', () => {
        this.reconnectAttempts = 0
        this.startHeartbeat()
        resolve()
      })

      this.ws.on('message', (data) => {
        this.handleMessage(data)
      })

      this.ws.on('close', () => {
        this.handleClose()
      })

      this.ws.on('error', (error) => {
        this.handleError(error)
      })
    })
  }

  private startHeartbeat(): void {
    this.heartbeatInterval = setInterval(() => {
      this.send({ type: 'ping' })
    }, 30000)  // 30 秒心跳
  }
}
```

### 重连逻辑

```typescript
private async handleClose(): Promise<void> {
  this.stopHeartbeat()

  if (this.reconnectAttempts < this.maxReconnectAttempts) {
    this.reconnectAttempts++
    const delay = Math.min(
      1000 * Math.pow(2, this.reconnectAttempts),
      30000  // 最大 30 秒
    )

    await sleep(delay)
    await this.reconnect()
  } else {
    this.emit('disconnected', { reason: 'max_reconnect_attempts' })
  }
}

private async reconnect(): Promise<void> {
  try {
    await this.connect(this.lastUrl, this.lastToken)
    this.emit('reconnected')
  } catch (error) {
    this.emit('reconnect_failed', error)
  }
}
```

## 消息适配

### SDK 消息格式

```typescript
// 入站消息（从 SDK）
export type SDKInboundMessage =
  | { type: 'user_message'; content: string }
  | { type: 'tool_result'; tool_use_id: string; content: string }
  | { type: 'interrupt' }
  | { type: 'permission_response'; request_id: string; decision: PermissionDecision }
  | { type: 'config_update'; settings: Partial<SettingsJson> }

// 出站消息（到 SDK）
export type SDKOutboundMessage =
  | { type: 'assistant_message'; content: ContentBlock[] }
  | { type: 'tool_use'; tool_use: ToolUseBlock }
  | { type: 'permission_request'; request: PermissionRequest }
  | { type: 'status_update'; status: SessionStatus }
  | { type: 'error'; message: string }
```

### 消息转换

```typescript
export class SDKMessageAdapter {
  // SDK 消息转内部格式
  fromSDKMessage(message: SDKInboundMessage): InternalMessage {
    switch (message.type) {
      case 'user_message':
        return createUserMessage({ content: message.content })

      case 'tool_result':
        return createToolResultMessage(message.tool_use_id, message.content)

      case 'permission_response':
        this.permissionResolver.resolve(message.request_id, message.decision)
        return null  // 不产生消息

      default:
        throw new Error(`Unknown SDK message type: ${message.type}`)
    }
  }

  // 内部格式转 SDK 消息
  toSDKMessage(message: Message): SDKOutboundMessage | null {
    switch (message.type) {
      case 'assistant':
        return {
          type: 'assistant_message',
          content: message.message.content,
        }

      case 'tool_use':
        return {
          type: 'tool_use',
          tool_use: message.toolUse,
        }

      default:
        return null
    }
  }
}
```

## 权限委托

### 远程权限请求

```typescript
export async function requestRemotePermission(
  request: PermissionRequest,
  session: RemoteSession,
): Promise<PermissionDecision> {
  const requestId = generateRequestId()

  // 发送权限请求到远程
  session.ws.send({
    type: 'permission_request',
    request: {
      id: requestId,
      toolName: request.toolName,
      input: sanitizeInput(request.input),
      message: request.message,
    },
  })

  // 等待远程响应
  return new Promise((resolve) => {
    const timeout = setTimeout(() => {
      resolve({ behavior: 'deny', message: 'Permission request timed out' })
    }, 60000)  // 60 秒超时

    session.permissionCallbacks.set(requestId, (decision) => {
      clearTimeout(timeout)
      resolve(decision)
    })
  })
}
```

## 会话恢复

### 恢复流程

```typescript
export async function resumeSession(sessionId: string): Promise<RemoteSession> {
  // 加载会话元数据
  const metadata = await loadSessionMetadata(sessionId)

  // 加载消息历史
  const messages = await loadSessionMessages(sessionId)

  // 恢复文件状态缓存
  const fileStateCache = await loadFileStateCache(sessionId)

  // 创建会话对象
  const session: RemoteSession = {
    id: sessionId,
    ...metadata,
    messages,
    fileStateCache,
    status: 'initialized',
  }

  return session
}
```

### 持久化

```typescript
export async function persistSession(session: RemoteSession): Promise<void> {
  // 保存元数据
  await saveSessionMetadata({
    id: session.id,
    name: session.name,
    cwd: session.cwd,
    createdAt: session.createdAt,
  })

  // 追加新消息
  for (const message of session.newMessages) {
    await appendSessionMessage(session.id, message)
  }

  // 保存文件状态
  await saveFileStateCache(session.id, session.fileStateCache)
}
```

## 活动追踪

### 会话活动信号

```typescript
export function sendSessionActivitySignal(): void {
  // 更新最后活动时间
  lastActivityTime = Date.now()

  // 发送心跳到远程
  if (isSessionActivityTrackingActive()) {
    void putWorkerHeartbeat()
  }
}

export function isSessionActivityTrackingActive(): boolean {
  return isEnvTruthy(process.env.CLAUDE_CODE_REMOTE)
}
```

### 空闲检测

```typescript
export function isSessionIdle(session: RemoteSession): boolean {
  const idleThreshold = 5 * 60 * 1000  // 5 分钟
  return Date.now() - session.lastActivityTime > idleThreshold
}
```

## 统计日志

```typescript
logEvent('tengu_remote_session_created', {
  sessionId,
  hasExistingHistory: messages.length > 0,
})

logEvent('tengu_remote_session_connected', {
  sessionId,
  connectionType: 'websocket',
})

logEvent('tengu_remote_session_message', {
  sessionId,
  messageType,
  direction: 'inbound' | 'outbound',
})
```

## 关键文件

- [src/remote/RemoteSessionManager.ts](src/remote/RemoteSessionManager.ts) — 会话管理
- [src/remote/SessionsWebSocket.ts](src/remote/SessionsWebSocket.ts) — WebSocket
- [src/remote/sdkMessageAdapter.ts](src/remote/sdkMessageAdapter.ts) — 消息适配
- [src/utils/sessionStorage.ts](src/utils/sessionStorage.ts) — 会话存储

## 关联文档

- [07-Bridge与远程/Bridge远程控制-Bridge-Remote-Control.md](Bridge远程控制-Bridge-Remote-Control.md)
- [02-核心引擎/消息处理流程-Message-Processing-Flow.md](../02-核心引擎/消息处理流程-Message-Processing-Flow.md)