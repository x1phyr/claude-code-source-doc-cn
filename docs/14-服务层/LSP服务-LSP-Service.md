# LSP 服务

## 概述

LSP (Language Server Protocol) 服务为 Claude Code 提供代码智能功能，包括诊断、悬停提示、跳转定义和自动补全等编辑器集成功能。

## 核心文件

- [src/services/lsp/manager.ts](src/services/lsp/manager.ts) — 服务管理器
- [src/services/lsp/LSPServerManager.ts](src/services/lsp/LSPServerManager.ts) — 服务器管理
- [src/services/lsp/LSPClient.ts](src/services/lsp/LSPClient.ts) — LSP 客户端
- [src/services/lsp/LSPDiagnosticRegistry.ts](src/services/lsp/LSPDiagnosticRegistry.ts) — 诊断注册

## 架构

```
Claude Code CLI
    ↓
LSP Server Manager
    ├── TypeScript Server
    ├── Python Server (pyright/pylance)
    ├── Go Server (gopls)
    └── Custom Servers
    ↓
LSP Client
    ├── 诊断订阅
    ├── 悬停请求
    ├── 定义跳转
    └── 补全请求
```

## 服务管理器

### 单例模式

```typescript
// 全局单例
let lspManagerInstance: LSPServerManager | undefined
let initializationState: 'not-started' | 'pending' | 'success' | 'failed' = 'not-started'

export function getLspServerManager(): LSPServerManager | undefined {
  if (initializationState === 'failed') {
    return undefined
  }
  return lspManagerInstance
}

export function getInitializationStatus():
  | { status: 'not-started' }
  | { status: 'pending' }
  | { status: 'success' }
  | { status: 'failed'; error: Error } {
  // 返回当前状态
}
```

### 初始化

```typescript
export function initializeLspServerManager(): void {
  // --bare 模式不启用 LSP
  if (isBareMode()) {
    return
  }

  // 跳过已初始化的实例
  if (lspManagerInstance !== undefined && initializationState !== 'failed') {
    return
  }

  // 创建管理器实例
  lspManagerInstance = createLSPServerManager()
  initializationState = 'pending'

  // 异步初始化（不阻塞启动）
  const currentGeneration = ++initializationGeneration

  initializationPromise = lspManagerInstance
    .initialize()
    .then(() => {
      if (currentGeneration === initializationGeneration) {
        initializationState = 'success'
        registerLSPNotificationHandlers(lspManagerInstance)
      }
    })
    .catch((error) => {
      if (currentGeneration === initializationGeneration) {
        initializationState = 'failed'
        initializationError = error
        lspManagerInstance = undefined
      }
    })
}
```

### 重新初始化

```typescript
export function reinitializeLspServerManager(): void {
  // 插件刷新后重新加载 LSP 配置
  if (initializationState === 'not-started') {
    return
  }

  // 关闭旧实例
  if (lspManagerInstance) {
    void lspManagerInstance.shutdown().catch(...)
  }

  // 重置状态
  lspManagerInstance = undefined
  initializationState = 'not-started'

  // 重新初始化
  initializeLspServerManager()
}
```

### 关闭

```typescript
export async function shutdownLspServerManager(): Promise<void> {
  if (lspManagerInstance === undefined) {
    return
  }

  try {
    await lspManagerInstance.shutdown()
  } catch (error) {
    logError(error)
  } finally {
    lspManagerInstance = undefined
    initializationState = 'not-started'
  }
}
```

## LSP Server Manager

### 服务器管理

```typescript
export interface LSPServerManager {
  // 初始化
  initialize(): Promise<void>

  // 关闭所有服务器
  shutdown(): Promise<void>

  // 获取所有服务器
  getAllServers(): Map<string, LSPServerInstance>

  // 获取特定语言的服务器
  getServerForLanguage(languageId: string): LSPServerInstance | undefined

  // 启动服务器
  startServer(name: string): Promise<void>

  // 停止服务器
  stopServer(name: string): Promise<void>
}
```

### 创建服务器管理器

```typescript
export function createLSPServerManager(): LSPServerManager {
  const servers = new Map<string, LSPServerInstance>()
  const diagnosticRegistry = createLSPDiagnosticRegistry()

  return {
    async initialize() {
      // 加载 LSP 配置
      const configs = await loadLSPConfigs()

      for (const [name, config] of Object.entries(configs)) {
        const instance = createLSPServerInstance(name, config)
        servers.set(name, instance)
      }
    },

    async shutdown() {
      for (const server of servers.values()) {
        await server.stop()
      }
      servers.clear()
    },

    getAllServers() {
      return servers
    },

    getServerForLanguage(languageId) {
      for (const server of servers.values()) {
        if (server.languages.includes(languageId)) {
          return server
        }
      }
      return undefined
    },

    async startServer(name) {
      const server = servers.get(name)
      if (server) {
        await server.start()
      }
    },

    async stopServer(name) {
      const server = servers.get(name)
      if (server) {
        await server.stop()
      }
    },
  }
}
```

## LSP Server Instance

### 服务器实例

```typescript
export interface LSPServerInstance {
  name: string
  languages: string[]
  state: 'stopped' | 'starting' | 'running' | 'error'
  process?: ChildProcess

  start(): Promise<void>
  stop(): Promise<void>
  restart(): Promise<void>

  sendRequest(method: string, params: unknown): Promise<unknown>
  sendNotification(method: string, params: unknown): void
}
```

### 创建实例

```typescript
export function createLSPServerInstance(
  name: string,
  config: LSPServerConfig,
): LSPServerInstance {
  let process: ChildProcess | undefined
  let state: LSPServerState = 'stopped'
  let client: LSPClient | undefined

  return {
    name,
    languages: config.languages,
    get state() { return state },

    async start() {
      if (state !== 'stopped') return

      state = 'starting'

      try {
        // 启动 LSP 服务器进程
        process = spawn(config.command, config.args ?? [], {
          cwd: config.cwd,
          env: { ...process.env, ...config.env },
        })

        // 创建 LSP 客户端
        client = createLSPClient(process.stdin, process.stdout)

        // 初始化握手
        await client.initialize({
          rootUri: config.rootUri,
          capabilities: getClientCapabilities(),
        })

        state = 'running'
      } catch (error) {
        state = 'error'
        throw error
      }
    },

    async stop() {
      if (client) {
        await client.shutdown()
      }
      if (process) {
        process.kill()
        process = undefined
      }
      state = 'stopped'
    },

    async sendRequest(method, params) {
      if (!client || state !== 'running') {
        throw new Error('Server not running')
      }
      return client.sendRequest(method, params)
    },

    sendNotification(method, params) {
      if (client && state === 'running') {
        client.sendNotification(method, params)
      }
    },
  }
}
```

## LSP Client

### 客户端实现

```typescript
export interface LSPClient {
  // 初始化
  initialize(params: InitializeParams): Promise<InitializeResult>

  // 关闭
  shutdown(): Promise<void>

  // 请求
  sendRequest(method: string, params: unknown): Promise<unknown>

  // 通知
  sendNotification(method: string, params: unknown): void

  // 事件订阅
  onNotification(method: string, handler: (params: unknown) => void): void
  onError(handler: (error: Error) => void): void
}
```

### LSP 请求

```typescript
// 文本悬停
async function hover(
  client: LSPClient,
  uri: string,
  position: Position,
): Promise<Hover | null> {
  return client.sendRequest('textDocument/hover', {
    textDocument: { uri },
    position,
  })
}

// 跳转定义
async function gotoDefinition(
  client: LSPClient,
  uri: string,
  position: Position,
): Promise<Location | Location[] | null> {
  return client.sendRequest('textDocument/definition', {
    textDocument: { uri },
    position,
  })
}

// 诊断
async function diagnostics(
  client: LSPClient,
  uri: string,
): Promise<Diagnostic[]> {
  return client.sendRequest('textDocument/diagnostic', {
    textDocument: { uri },
  })
}

// 补全
async function completion(
  client: LSPClient,
  uri: string,
  position: Position,
): Promise<CompletionItem[] | null> {
  return client.sendRequest('textDocument/completion', {
    textDocument: { uri },
    position,
  })
}
```

## 诊断注册

### Diagnostic Registry

```typescript
export interface LSPDiagnosticRegistry {
  // 获取诊断
  getDiagnostics(uri: string): Diagnostic[]

  // 获取所有诊断
  getAllDiagnostics(): Map<string, Diagnostic[]>

  // 清除诊断
  clearDiagnostics(uri: string): void

  // 事件处理
  handleDiagnostics(params: PublishDiagnosticsParams): void
}

export function createLSPDiagnosticRegistry(): LSPDiagnosticRegistry {
  const diagnostics = new Map<string, Diagnostic[]>()

  return {
    getDiagnostics(uri) {
      return diagnostics.get(uri) ?? []
    },

    getAllDiagnostics() {
      return diagnostics
    },

    clearDiagnostics(uri) {
      diagnostics.delete(uri)
    },

    handleDiagnostics(params) {
      if (params.diagnostics.length === 0) {
        diagnostics.delete(params.uri)
      } else {
        diagnostics.set(params.uri, params.diagnostics)
      }
    },
  }
}
```

### 被动反馈

```typescript
export function registerLSPNotificationHandlers(
  manager: LSPServerManager,
): void {
  for (const server of manager.getAllServers().values()) {
    server.onNotification('textDocument/publishDiagnostics', (params) => {
      manager.diagnosticRegistry.handleDiagnostics(params)
    })
  }
}
```

## 配置加载

### LSP 配置格式

```json
{
  "lspServers": {
    "typescript": {
      "command": "typescript-language-server",
      "args": ["--stdio"],
      "languages": ["typescript", "javascript", "typescriptreact", "javascriptreact"]
    },
    "python": {
      "command": "pyright-langserver",
      "args": ["--stdio"],
      "languages": ["python"]
    },
    "go": {
      "command": "gopls",
      "languages": ["go"]
    }
  }
}
```

### 加载配置

```typescript
export async function loadLSPConfigs(): Promise<Record<string, LSPServerConfig>> {
  const configs: Record<string, LSPServerConfig> = {}

  // 从设置文件加载
  const settings = getSettings()
  if (settings.lspServers) {
    Object.assign(configs, settings.lspServers)
  }

  // 从插件加载
  const plugins = await loadAllPlugins()
  for (const plugin of plugins) {
    if (plugin.lspServers) {
      Object.assign(configs, plugin.lspServers)
    }
  }

  return configs
}
```

## 连接检查

```typescript
export function isLspConnected(): boolean {
  if (initializationState === 'failed') return false

  const manager = getLspServerManager()
  if (!manager) return false

  const servers = manager.getAllServers()
  if (servers.size === 0) return false

  for (const server of servers.values()) {
    if (server.state !== 'error') return true
  }

  return false
}
```

## 工具集成

### LSPTool

```typescript
export class LSPTool implements Tool {
  name = 'LSP'

  async call(args: LSPToolInput, context: ToolUseContext): Promise<ToolResult> {
    const manager = getLspServerManager()
    if (!manager) {
      throw new Error('LSP not initialized')
    }

    const server = manager.getServerForLanguage(args.language)
    if (!server) {
      throw new Error(`No LSP server for language: ${args.language}`)
    }

    if (server.state !== 'running') {
      await server.start()
    }

    switch (args.action) {
      case 'hover':
        const hoverResult = await hover(server.client, args.uri, args.position)
        return { output: hoverResult }

      case 'definition':
        const defResult = await gotoDefinition(server.client, args.uri, args.position)
        return { output: defResult }

      case 'diagnostics':
        const diagResult = await diagnostics(server.client, args.uri)
        return { output: diagResult }

      default:
        throw new Error(`Unknown LSP action: ${args.action}`)
    }
  }
}
```

## 统计日志

```typescript
logEvent('tengu_lsp_server_started', {
  server_name: name,
  languages: config.languages.join(','),
})

logEvent('tengu_lsp_server_error', {
  server_name: name,
  error_message: errorMessage(error),
})

logEvent('tengu_lsp_request', {
  server_name: name,
  method: method,
  duration_ms: endTime - startTime,
})
```

## 关键文件

- [src/services/lsp/manager.ts](src/services/lsp/manager.ts) — 服务管理
- [src/services/lsp/LSPServerManager.ts](src/services/lsp/LSPServerManager.ts) — 服务器管理
- [src/services/lsp/LSPClient.ts](src/services/lsp/LSPClient.ts) — LSP 客户端
- [src/services/lsp/types.ts](src/services/lsp/types.ts) — 类型定义

## 关联文档

- [14-服务层/服务层详解-Services-Detail.md](服务层详解-Services-Detail.md)
- [03-工具系统/工具系统详解-Tool-System.md](../03-工具系统/工具系统详解-Tool-System.md)