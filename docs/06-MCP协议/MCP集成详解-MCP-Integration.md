# MCP 集成详解

## 概述

MCP (Model Context Protocol) 是一个开放协议，允许 Claude Code 连接外部工具和资源。通过 MCP，Claude 可以访问数据库、API、文件系统等各种能力。

## 传输类型

### 支持的传输

```typescript
type Transport =
  | 'stdio'      // 标准输入输出
  | 'sse'        // Server-Sent Events
  | 'sse-ide'    // IDE 专用 SSE
  | 'http'       // HTTP
  | 'ws'         // WebSocket
  | 'sdk'        // 内嵌 SDK
```

### 配置结构

#### stdio 配置

```typescript
type McpStdioServerConfig = {
  type?: 'stdio'
  command: string           // 命令
  args?: string[]           // 参数
  env?: Record<string, string>  // 环境变量
}
```

#### SSE 配置

```typescript
type McpSSEServerConfig = {
  type: 'sse'
  url: string               // SSE 端点 URL
  headers?: Record<string, string>  // 请求头
  headersHelper?: string    // 请求头辅助
  oauth?: McpOAuthConfig    // OAuth 配置
}
```

#### HTTP 配置

```typescript
type McpHTTPServerConfig = {
  type: 'http'
  url: string
  headers?: Record<string, string>
  headersHelper?: string
  oauth?: McpOAuthConfig
}
```

#### WebSocket 配置

```typescript
type McpWebSocketServerConfig = {
  type: 'ws'
  url: string
  headers?: Record<string, string>
  headersHelper?: string
}
```

#### SDK 配置

```typescript
type McpSdkServerConfig = {
  type: 'sdk'
  name: string              // SDK 名称
}
```

## 服务器连接状态

### 连接类型

```typescript
type MCPServerConnection =
  | ConnectedMCPServer      // 已连接
  | FailedMCPServer         // 连接失败
  | NeedsAuthMCPServer      // 需要认证
  | PendingMCPServer        // 等待中
  | DisabledMCPServer       // 已禁用
```

### 已连接服务器

```typescript
type ConnectedMCPServer = {
  client: Client                    // MCP 客户端实例
  name: string                      // 服务器名称
  type: 'connected'
  capabilities: ServerCapabilities  // 服务器能力
  serverInfo?: {
    name: string
    version: string
  }
  instructions?: string             // 服务器说明
  config: ScopedMcpServerConfig     // 配置
  cleanup: () => Promise<void>      // 清理函数
}
```

## 配置作用域

```typescript
type ConfigScope =
  | 'local'       // 本地配置
  | 'user'        // 用户级配置
  | 'project'     // 项目级配置
  | 'dynamic'     // 动态配置
  | 'enterprise'  // 企业配置
  | 'claudeai'    // Claude.ai 配置
  | 'managed'     // 托管配置
```

## 配置格式

### mcp.json 示例

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/allowed/dir"]
    },
    "github": {
      "type": "http",
      "url": "https://api.github.com/mcp",
      "headers": {
        "Authorization": "Bearer ${GITHUB_TOKEN}"
      }
    },
    "database": {
      "type": "sse",
      "url": "http://localhost:3000/sse",
      "oauth": {
        "clientId": "my-client-id",
        "authServerMetadataUrl": "https://auth.example.com/.well-known/oauth-authorization-server"
      }
    }
  }
}
```

### 配置位置

| 作用域 | 文件位置 |
|--------|----------|
| user | `~/.claude/settings.json` |
| project | `.claude/settings.json` |
| local | `.claude/settings.local.json` |

## OAuth 认证

### OAuth 配置

```typescript
type McpOAuthConfig = {
  clientId?: string                  // 客户端 ID
  callbackPort?: number              // 回调端口
  authServerMetadataUrl?: string     // 授权服务器元数据 URL
  xaa?: boolean                      // 跨应用访问
}
```

### 认证流程

```
1. 检测需要认证 (NeedsAuthMCPServer)
    ↓
2. 启动本地回调服务器
    ↓
3. 打开浏览器访问授权页面
    ↓
4. 用户授权后回调
    ↓
5. 获取访问令牌
    ↓
6. 连接 MCP 服务器
```

## 工具发现

### MCP 工具注册

MCP 服务器提供的工具会被注册到 Claude Code 的工具系统：

```typescript
type MCPTool = Tool & {
  isMcp: true
  mcpInfo: {
    serverName: string
    toolName: string
  }
}
```

### 工具名称规范化

MCP 工具名称会被规范化：

```
原始名称: get_user
服务器: myserver
规范化: mcp__myserver__get_user
```

## 资源访问

### 资源类型

```typescript
type ServerResource = Resource & {
  server: string    // 来源服务器
}
```

### 资源操作

- `ListMcpResourcesTool` - 列出资源
- `ReadMcpResourceTool` - 读取资源

## 核心模块

### 文件结构

```
src/services/mcp/
├── client.ts           # MCP 客户端
├── config.ts           # 配置管理
├── types.ts            # 类型定义
├── auth.ts             # 认证逻辑
├── oauthPort.ts        # OAuth 端口管理
├── normalization.ts    # 名称规范化
├── utils.ts            # 工具函数
├── InProcessTransport.ts   # 进程内传输
├── SdkControlTransport.ts  # SDK 控制传输
├── useManageMCPConnections.ts  # 连接管理 hook
├── elicitationHandler.ts  # Elicitation 处理
├── officialRegistry.ts    # 官方注册表
├── xaa.ts                # 跨应用访问
├── xaaIdpLogin.ts        # XAA IdP 登录
├── channelAllowlist.ts   # 渠道白名单
├── channelPermissions.ts # 渠道权限
└── channelNotification.ts # 渠道通知
```

### 客户端管理

```typescript
// useManageMCPConnections.ts
function useManageMCPConnections() {
  // 连接所有配置的 MCP 服务器
  // 管理连接状态
  // 处理重连逻辑
  // 提供工具列表
}
```

## 错误处理

### 连接失败

```typescript
type FailedMCPServer = {
  name: string
  type: 'failed'
  config: ScopedMcpServerConfig
  error?: string           // 错误信息
}
```

### 重连机制

```typescript
type PendingMCPServer = {
  name: string
  type: 'pending'
  config: ScopedMcpServerConfig
  reconnectAttempt?: number       // 当前尝试次数
  maxReconnectAttempts?: number   // 最大尝试次数
}
```

## MCP 命令

### /mcp 命令

提供 MCP 管理界面：
- 列出服务器状态
- 添加/删除服务器
- 重新连接服务器
- 查看服务器详情

## 关键文件

- `src/services/mcp/types.ts` - MCP 类型定义
- `src/services/mcp/client.ts` - MCP 客户端
- `src/services/mcp/config.ts` - 配置管理
- `src/commands/mcp/` - MCP 命令实现

## 关联文档

- [05-工具系统](05-tool-system.md)
- [06-命令系统](06-command-system.md)
- [23-配置参考](23-configuration-reference.md)