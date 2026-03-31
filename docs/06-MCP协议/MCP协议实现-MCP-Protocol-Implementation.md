# MCP 协议实现

## 概述

MCP (Model Context Protocol) 是开放协议，允许 Claude 连接外部工具和资源。Claude Code 实现了完整的 MCP 客户端，支持多种传输类型和 OAuth 认证。

## 核心文件

- [src/services/mcp/client.ts](src/services/mcp/client.ts) — MCP 客户端
- [src/services/mcp/types.ts](src/services/mcp/types.ts) — 类型定义
- [src/services/mcp/auth.ts](src/services/mcp/auth.ts) — OAuth 认证
- [src/tools/MCPTool/MCPTool.ts](src/tools/MCPTool/MCPTool.ts) — MCP 工具

## 传输类型

### stdio 传输

```typescript
import { StdioClientTransport } from '@modelcontextprotocol/sdk/client/stdio.js'

const transport = new StdioClientTransport({
  command: config.command,
  args: config.args,
  env: { ...process.env, ...config.env },
})
```

**配置格式**：
```json
{
  "type": "stdio",
  "command": "npx",
  "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path"],
  "env": { "API_KEY": "..." }
}
```

### SSE 传输

```typescript
import { SSEClientTransport } from '@modelcontextprotocol/sdk/client/sse.js'

const transport = new SSEClientTransport(
  new URL(config.url),
  {
    headers: config.headers,
    // OAuth 支持
    authProvider: claudeAuthProvider,
  }
)
```

### HTTP 传输 (Streamable HTTP)

```typescript
import { StreamableHTTPClientTransport } from '@modelcontextprotocol/sdk/client/streamableHttp.js'

const transport = new StreamableHTTPClientTransport(
  new URL(config.url),
  {
    headers: config.headers,
    // MCP Streamable HTTP spec 要求
    requestInit: {
      headers: { Accept: 'application/json, text/event-stream' }
    }
  }
)
```

### WebSocket 传输

```typescript
import { WebSocketTransport } from '../../utils/mcpWebSocketTransport.js'

const wsClient = await createNodeWsClient(url, options)
const transport = new WebSocketTransport(wsClient)
```

### SDK 内嵌传输

```typescript
import { SdkControlClientTransport } from './SdkControlTransport.js'

// 内嵌 SDK 控制传输
const transport = new SdkControlClientTransport(...)
```

## 客户端连接

### 连接管理

```typescript
export interface MCPServerConnection =
  | ConnectedMCPServer
  | FailedMCPServer
  | NeedsAuthMCPServer
  | PendingMCPServer
  | DisabledMCPServer

export type ConnectedMCPServer = {
  client: Client
  name: string
  type: 'connected'
  capabilities: ServerCapabilities
  serverInfo?: { name: string; version: string }
  instructions?: string
  config: ScopedMcpServerConfig
  cleanup: () => Promise<void>
}
```

### 连接流程

```typescript
async function connectMcpServer(
  name: string,
  config: ScopedMcpServerConfig,
): Promise<MCPServerConnection> {
  // 1. 创建传输
  const transport = await createTransport(config)

  // 2. 创建客户端
  const client = new Client({ name: 'claude-code', version: VERSION })

  // 3. 连接
  await client.connect(transport)

  // 4. 获取能力
  const capabilities = client.getServerCapabilities()

  // 5. 发现工具
  const tools = await discoverTools(client)

  return {
    client,
    name,
    type: 'connected',
    capabilities,
    config,
    cleanup: async () => {
      await client.close()
    }
  }
}
```

## 工具发现

### 列出工具

```typescript
async function discoverTools(client: Client): Promise<ListToolsResult> {
  const result = await client.request(
    { method: 'tools/list' },
    ListToolsResultSchema,
  )
  return result
}
```

### 工具名称转换

```typescript
export function buildMcpToolName(serverName: string, toolName: string): string {
  return `mcp__${serverName}__${toolName}`
}

export function mcpInfoFromString(name: string): McpToolInfo | null {
  const match = name.match(/^mcp__(.+)__(.+)$/)
  if (!match) return null
  return { serverName: match[1], toolName: match[2] }
}
```

## 工具调用

### MCPTool 实现

```typescript
export class MCPTool implements Tool {
  name: string
  inputSchema: z.ZodType
  mcpInfo: { serverName: string; toolName: string }

  async call(args, context, canUseTool, parentMessage, onProgress) {
    const client = await ensureConnectedClient(this.mcpInfo.serverName)

    const result = await client.request(
      {
        method: 'tools/call',
        params: {
          name: this.mcpInfo.toolName,
          arguments: args,
        }
      },
      CallToolResultSchema,
    )

    return {
      output: processMcpResult(result),
      metadata: { _meta: result._meta }
    }
  }
}
```

### 进度报告

```typescript
export type MCPProgress = {
  type: 'progress'
  progressToken: string | number
  progress: number
  total?: number
}

// 工具调用时传递进度回调
onProgress?.({
  type: 'progress',
  progressToken: token,
  progress: current,
  total: total,
})
```

## 资源访问

### 列出资源

```typescript
async function listResources(client: Client): Promise<ResourceLink[]> {
  const result = await client.request(
    { method: 'resources/list' },
    ListResourcesResultSchema,
  )
  return result.resources
}
```

### 读取资源

```typescript
async function readResource(
  client: Client,
  uri: string,
): Promise<ReadResourceResult> {
  const result = await client.request(
    { method: 'resources/read', params: { uri } },
    ReadResourceResultSchema,
  )
  return result
}
```

## 错误处理

### 认证错误

```typescript
export class McpAuthError extends Error {
  serverName: string
  constructor(serverName: string, message: string) {
    super(message)
    this.name = 'McpAuthError'
    this.serverName = serverName
  }
}

// 401 错误处理
if (response.status === 401) {
  setMcpAuthCacheEntry(name)
  return { name, type: 'needs-auth', config }
}
```

### 会话过期

```typescript
export function isMcpSessionExpiredError(error: Error): boolean {
  const httpStatus = (error as any).code
  if (httpStatus !== 404) return false

  // MCP 服务器返回: {"error":{"code":-32001,"message":"Session not found"}}
  return (
    error.message.includes('"code":-32001') ||
    error.message.includes('"code": -32001')
  )
}
```

### 工具调用错误

```typescript
export class McpToolCallError extends Error {
  constructor(
    message: string,
    telemetryMessage: string,
    readonly mcpMeta?: { _meta?: Record<string, unknown> },
  ) {
    super(message)
    this.name = 'McpToolCallError'
  }
}

// 工具返回 isError: true
if (result.isError) {
  throw new McpToolCallError(
    processContent(result.content),
    'MCP tool error',
    { _meta: result._meta }
  )
}
```

## 超时配置

```typescript
// 默认 MCP 工具超时（约 27.8 小时）
const DEFAULT_MCP_TOOL_TIMEOUT_MS = 100_000_000

// 连接超时
function getConnectionTimeoutMs(): number {
  return parseInt(process.env.MCP_TIMEOUT || '', 10) || 30000
}

// 单个请求超时
const MCP_REQUEST_TIMEOUT_MS = 60000

// OAuth 请求超时
const AUTH_REQUEST_TIMEOUT_MS = 30000
```

## 认证缓存

```typescript
const MCP_AUTH_CACHE_TTL_MS = 15 * 60 * 1000 // 15 分钟

type McpAuthCacheData = Record<string, { timestamp: number }>

async function isMcpAuthCached(serverId: string): Promise<boolean> {
  const cache = await getMcpAuthCache()
  const entry = cache[serverId]
  if (!entry) return false
  return Date.now() - entry.timestamp < MCP_AUTH_CACHE_TTL_MS
}
```

## 描述截断

```typescript
// MCP 工具描述最大长度
const MAX_MCP_DESCRIPTION_LENGTH = 2048

// OpenAPI 生成的 MCP 服务器可能输出 15-60KB 的描述
// 截断以避免上下文膨胀
function truncateDescription(description: string): string {
  if (description.length <= MAX_MCP_DESCRIPTION_LENGTH) {
    return description
  }
  return description.slice(0, MAX_MCP_DESCRIPTION_LENGTH) + '...'
}
```

## 图像处理

```typescript
const IMAGE_MIME_TYPES = new Set([
  'image/jpeg',
  'image/png',
  'image/gif',
  'image/webp',
])

async function processImageContent(content: ImageContent): Promise<ImageBlock> {
  // 调整大小和下采样
  const buffer = Buffer.from(content.data, 'base64')
  const resized = await maybeResizeAndDownsampleImageBuffer(buffer)
  return {
    type: 'image',
    source: {
      type: 'base64',
      media_type: content.mimeType,
      data: resized.toString('base64'),
    }
  }
}
```

## 统计事件

```typescript
logEvent('tengu_mcp_server_connected', {
  transportType,
  toolCount: tools.length,
  hasResources: !!capabilities.resources,
  hasPrompts: !!capabilities.prompts,
})

logEvent('tengu_mcp_tool_call', {
  serverName,
  toolName,
  durationMs,
  success: true,
})

logEvent('tengu_mcp_server_needs_auth', {
  transportType,
  mcpServerBaseUrl,
})
```

## 关键文件

- [src/services/mcp/client.ts](src/services/mcp/client.ts) — 客户端实现
- [src/services/mcp/types.ts](src/services/mcp/types.ts) — 类型定义
- [src/services/mcp/auth.ts](src/services/mcp/auth.ts) — OAuth 认证
- [src/services/mcp/config.ts](src/services/mcp/config.ts) — 配置管理
- [src/tools/MCPTool/MCPTool.ts](src/tools/MCPTool/MCPTool.ts) — 工具实现

## 关联文档

- [06-MCP协议/MCP集成详解-MCP-Integration.md](MCP集成详解-MCP-Integration.md)
- [06-MCP协议/OAuth认证流程-OAuth-Authentication-Flow.md](OAuth认证流程-OAuth-Authentication-Flow.md)