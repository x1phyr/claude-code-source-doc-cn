# Bridge 远程控制

## 概述

Bridge 是 Claude Code 的远程控制模式，允许从 claude.ai 网站或 IDE 扩展远程控制本地 CLI 会话，实现跨设备的协作开发。

## 核心文件

- [src/bridge/bridgeMain.ts](src/bridge/bridgeMain.ts) — 主循环
- [src/bridge/replBridge.ts](src/bridge/replBridge.ts) — Bridge 连接
- [src/bridge/types.ts](src/bridge/types.ts) — 类型定义

## 架构

```
claude.ai / IDE
    ↓ WebSocket/SSE
Bridge Loop (本地)
    ├── 会话管理
    ├── 消息转发
    └── 权限验证
    ↓
本地 CLI 会话
```

## Bridge 模式

### 启动 Bridge

```typescript
// 通过 /bridge 命令启动
export async function startBridge(config: BridgeConfig): Promise<void> {
  // 1. 获取环境凭证
  const { environmentId, environmentSecret } = await getBridgeCredentials()

  // 2. 创建 API 客户端
  const api = createBridgeApiClient(config)

  // 3. 创建会话启动器
  const spawner = createSessionSpawner()

  // 4. 运行 Bridge 循环
  await runBridgeLoop(
    config,
    environmentId,
    environmentSecret,
    api,
    spawner,
    logger,
    signal,
  )
}
```

### Bridge 配置

```typescript
export interface BridgeConfig {
  // 远程服务 URL
  remoteSessionUrl: string

  // 会话超时
  sessionTimeoutMs: number

  // 最大并发会话数
  maxSessions: number

  // 轮询间隔
  pollIntervalMs: number

  // 心跳间隔
  heartbeatIntervalMs: number
}

export const DEFAULT_SESSION_TIMEOUT_MS = 30 * 60 * 1000  // 30 分钟
```

## 主循环

### Bridge Loop

```typescript
export async function runBridgeLoop(
  config: BridgeConfig,
  environmentId: string,
  environmentSecret: string,
  api: BridgeApiClient,
  spawner: SessionSpawner,
  logger: BridgeLogger,
  signal: AbortSignal,
): Promise<void> {
  const activeSessions = new Map<string, SessionHandle>()
  const sessionStartTimes = new Map<string, number>()
  const capacityWake = createCapacityWake(signal)

  while (!signal.aborted) {
    try {
      // 1. 轮询新工作
      const work = await api.pollWork({
        environmentId,
        environmentSecret,
        activeSessionIds: [...activeSessions.keys()],
      })

      // 2. 处理新会话请求
      for (const item of work.newItems) {
        if (activeSessions.size >= config.maxSessions) {
          // 达到容量上限，等待
          await capacityWake.wait()
        }

        const handle = spawner.spawn({
          sessionId: item.sessionId,
          sdkUrl: item.sdkUrl,
        })

        activeSessions.set(item.sessionId, handle)
      }

      // 3. 心跳活跃会话
      await heartbeatSessions(activeSessions, api)

      // 4. 清理完成的会话
      cleanupCompletedSessions(activeSessions)

    } catch (error) {
      await handleBridgeError(error, backoff)
    }

    await sleep(config.pollIntervalMs)
  }
}
```

## 会话管理

### 会话启动器

```typescript
export interface SessionSpawner {
  spawn(opts: SessionSpawnOpts, dir: string): SessionHandle
}

export interface SessionSpawnOpts {
  sessionId: string
  sdkUrl: string
  worktreePath?: string
  initialPrompt?: string
}

export interface SessionHandle {
  sessionId: string
  process: ChildProcess
  accessToken: string
  cleanup: () => Promise<void>
}

export function createSessionSpawner(): SessionSpawner {
  return {
    spawn(opts: SessionSpawnOpts, dir: string): SessionHandle {
      const process = spawnChildClaude({
        sdkUrl: opts.sdkUrl,
        sessionId: opts.sessionId,
        cwd: dir,
      })

      return {
        sessionId: opts.sessionId,
        process,
        accessToken: '',
        cleanup: async () => {
          process.kill('SIGTERM')
          await waitForExit(process, 30000)
        },
      }
    },
  }
}
```

### 子进程启动

```typescript
function spawnChildClaude(opts: {
  sdkUrl: string
  sessionId: string
  cwd: string
}): ChildProcess {
  const args = [
    ...spawnScriptArgs(),
    '--sdk-url', opts.sdkUrl,
    '--session-id', opts.sessionId,
  ]

  return spawn(process.execPath, args, {
    cwd: opts.cwd,
    stdio: ['ignore', 'pipe', 'pipe'],
    env: {
      ...process.env,
      CLAUDE_CODE_REMOTE: 'true',
    },
  })
}
```

## 心跳机制

### 心跳请求

```typescript
async function heartbeatSessions(
  sessions: Map<string, SessionHandle>,
  api: BridgeApiClient,
): Promise<void> {
  for (const [sessionId, handle] of sessions) {
    try {
      await api.heartbeat({
        sessionId,
        accessToken: handle.accessToken,
      })
    } catch (error) {
      if (isExpiredErrorType(error)) {
        // Token 过期，重新连接
        await reconnectSession(sessionId, handle)
      }
    }
  }
}
```

### 容量唤醒

```typescript
export function createCapacityWake(signal: AbortSignal): CapacityWake {
  let resolveWait: (() => void) | null = null

  return {
    wait: () => new Promise(resolve => {
      resolveWait = resolve
    }),
    wake: () => {
      resolveWait?.()
      resolveWait = null
    },
  }
}
```

## 消息传递

### 入站消息

```typescript
// 从远程接收消息
export async function handleInboundMessage(
  message: BridgeInboundMessage,
  context: BridgeContext,
): Promise<void> {
  switch (message.type) {
    case 'user_message':
      // 转发用户消息到 CLI
      await context.session.sendUserMessage(message.content)
      break

    case 'tool_result':
      // 转发工具结果
      await context.session.sendToolResult(message.toolUseId, message.result)
      break

    case 'interrupt':
      // 中断当前操作
      context.session.interrupt()
      break

    case 'permission_decision':
      // 权限决策响应
      context.permissionCallbacks.resolve(message.requestId, message.decision)
      break
  }
}
```

### 出站消息

```typescript
// 发送消息到远程
export async function sendOutboundMessage(
  message: BridgeOutboundMessage,
  api: BridgeApiClient,
): Promise<void> {
  await api.sendMessage({
    type: message.type,
    sessionId: message.sessionId,
    payload: message.payload,
  })
}

export type BridgeOutboundMessage =
  | { type: 'assistant_message'; content: ContentBlock[] }
  | { type: 'tool_use'; toolUse: ToolUseBlock }
  | { type: 'permission_request'; requestId: string; toolName: string }
  | { type: 'session_done'; status: SessionDoneStatus }
```

## 权限委托

### 权限请求转发

```typescript
// 本地 CLI 请求权限时，转发到远程
export async function requestRemotePermission(
  toolName: string,
  input: unknown,
  context: BridgeContext,
): Promise<PermissionDecision> {
  const requestId = generateRequestId()

  // 发送权限请求到远程
  await sendOutboundMessage({
    type: 'permission_request',
    sessionId: context.sessionId,
    payload: {
      requestId,
      toolName,
      input: sanitizeForRemote(input),
    },
  }, context.api)

  // 等待远程决策
  return new Promise(resolve => {
    context.permissionCallbacks.register(requestId, resolve)
  })
}
```

## 错误处理

### 退避策略

```typescript
export type BackoffConfig = {
  connInitialMs: number      // 初始连接退避
  connCapMs: number          // 连接退避上限
  connGiveUpMs: number       // 连接放弃阈值
  generalInitialMs: number   // 一般错误初始退避
  generalCapMs: number       // 一般错误退避上限
  generalGiveUpMs: number    // 一般错误放弃阈值
}

const DEFAULT_BACKOFF: BackoffConfig = {
  connInitialMs: 2_000,
  connCapMs: 120_000,      // 2 分钟
  connGiveUpMs: 600_000,   // 10 分钟
  generalInitialMs: 500,
  generalCapMs: 30_000,
  generalGiveUpMs: 600_000,
}
```

### 错误分类

```typescript
async function handleBridgeError(
  error: unknown,
  backoff: BackoffState,
): Promise<void> {
  if (isAuthError(error)) {
    // 认证错误，尝试刷新 token
    await refreshToken()
    return
  }

  if (isNetworkError(error)) {
    // 网络错误，指数退避
    await sleep(backoff.nextDelay())
    return
  }

  if (isFatalError(error)) {
    // 致命错误，退出
    throw new BridgeFatalError(error.message)
  }

  // 其他错误，记录并继续
  logError(error)
}
```

## 统计日志

```typescript
logEvent('tengu_bridge_session_start', {
  sessionId,
  environmentId,
  spawnMode,
})

logEvent('tengu_bridge_session_done', {
  sessionId,
  status,
  durationMs,
  toolsExecuted,
})

logEvent('tengu_bridge_heartbeat', {
  activeSessions,
  capacityReached,
})
```

## 关键文件

- [src/bridge/bridgeMain.ts](src/bridge/bridgeMain.ts) — 主循环
- [src/bridge/replBridge.ts](src/bridge/replBridge.ts) — 连接管理
- [src/bridge/types.ts](src/bridge/types.ts) — 类型定义
- [src/bridge/bridgeApi.ts](src/bridge/bridgeApi.ts) — API 客户端
- [src/commands/bridge/bridge.tsx](src/commands/bridge/bridge.tsx) — 命令入口

## 关联文档

- [07-Bridge与远程/远程会话管理-Remote-Session-Management.md](远程会话管理-Remote-Session-Management.md)