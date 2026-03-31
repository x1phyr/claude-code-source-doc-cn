# MCP 服务器开发

## 概述

本文档介绍如何为 Claude Code 开发自定义 MCP (Model Context Protocol) 服务器，扩展 Claude Code 的工具能力。

## 核心文件

- [src/services/mcp/client.ts](src/services/mcp/client.ts) — MCP 客户端
- [src/services/mcp/types.ts](src/services/mcp/types.ts) — 类型定义
- [src/tools/MCPTool/MCPTool.ts](src/tools/MCPTool/MCPTool.ts) — MCP 工具

## MCP 协议基础

### 传输类型

```typescript
type MCPTransport =
  | { type: 'stdio'; command: string; args?: string[] }
  | { type: 'sse'; url: string }
  | { type: 'http'; url: string }
  | { type: 'websocket'; url: string }
```

### 消息格式

```typescript
// 请求
interface JSONRPCRequest {
  jsonrpc: '2.0'
  id: number | string
  method: string
  params?: unknown
}

// 响应
interface JSONRPCResponse {
  jsonrpc: '2.0'
  id: number | string
  result?: unknown
  error?: { code: number; message: string; data?: unknown }
}

// 通知
interface JSONRPCNotification {
  jsonrpc: '2.0'
  method: string
  params?: unknown
}
```

## 创建 MCP 服务器

### 基本结构

```typescript
// my-mcp-server/index.ts
import { Server } from '@modelcontextprotocol/sdk/server/index.js'
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js'

const server = new Server(
  { name: 'my-mcp-server', version: '1.0.0' },
  { capabilities: { tools: {} } }
)

// 注册工具列表
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: 'my_tool',
        description: 'My custom tool description',
        inputSchema: {
          type: 'object',
          properties: {
            input: { type: 'string', description: 'Input parameter' },
          },
          required: ['input'],
        },
      },
    ],
  }
})

// 处理工具调用
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params

  if (name === 'my_tool') {
    const result = await processInput(args.input)
    return {
      content: [{ type: 'text', text: result }],
    }
  }

  throw new Error(`Unknown tool: ${name}`)
})

// 启动服务器
const transport = new StdioServerTransport()
await server.connect(transport)
```

### 使用 TypeScript

```typescript
import { Server } from '@modelcontextprotocol/sdk/server/index.js'
import {
  ListToolsRequestSchema,
  CallToolRequestSchema,
  type Tool,
} from '@modelcontextprotocol/sdk/types.js'

// 定义工具
const tools: Tool[] = [
  {
    name: 'echo',
    description: 'Echo back the input',
    inputSchema: {
      type: 'object',
      properties: {
        message: { type: 'string' },
      },
      required: ['message'],
    },
  },
  {
    name: 'add',
    description: 'Add two numbers',
    inputSchema: {
      type: 'object',
      properties: {
        a: { type: 'number' },
        b: { type: 'number' },
      },
      required: ['a', 'b'],
    },
  },
]

// 工具实现
const toolHandlers: Record<string, (args: unknown) => Promise<string>> = {
  echo: async (args: { message: string }) => args.message,
  add: async (args: { a: number; b: number }) => String(args.a + args.b),
}

const server = new Server(
  { name: 'my-server', version: '1.0.0' },
  { capabilities: { tools: {} } }
)

server.setRequestHandler(ListToolsRequestSchema, async () => ({ tools }))

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params
  const handler = toolHandlers[name]

  if (!handler) {
    throw new Error(`Unknown tool: ${name}`)
  }

  const result = await handler(args)
  return {
    content: [{ type: 'text', text: result }],
  }
})
```

## 配置 MCP 服务器

### 项目级配置

```json
// .mcp.json
{
  "mcpServers": {
    "my-server": {
      "type": "stdio",
      "command": "node",
      "args": ["./mcp-servers/my-server/dist/index.js"],
      "env": {
        "API_KEY": "${MY_API_KEY}"
      }
    }
  }
}
```

### 全局配置

```json
// ~/.claude/settings.json
{
  "mcpServers": {
    "global-server": {
      "command": "npx",
      "args": ["-y", "@my-org/mcp-server"]
    }
  }
}
```

## 高级功能

### 资源支持

```typescript
// 注册资源
server.setRequestHandler(ListResourcesRequestSchema, async () => {
  return {
    resources: [
      {
        uri: 'file:///config.json',
        name: 'Configuration',
        mimeType: 'application/json',
      },
    ],
  }
})

// 读取资源
server.setRequestHandler(ReadResourceRequestSchema, async (request) => {
  const { uri } = request.params

  if (uri === 'file:///config.json') {
    const config = await loadConfig()
    return {
      contents: [
        {
          uri,
          mimeType: 'application/json',
          text: JSON.stringify(config),
        },
      ],
    }
  }

  throw new Error(`Unknown resource: ${uri}`)
})
```

### Prompt 模板

```typescript
server.setRequestHandler(ListPromptsRequestSchema, async () => {
  return {
    prompts: [
      {
        name: 'analyze-code',
        description: 'Analyze code for issues',
        arguments: [
          {
            name: 'language',
            description: 'Programming language',
            required: true,
          },
        ],
      },
    ],
  }
})

server.setRequestHandler(GetPromptRequestSchema, async (request) => {
  const { name, arguments: args } = request.params

  if (name === 'analyze-code') {
    return {
      description: 'Code analysis prompt',
      messages: [
        {
          role: 'user',
          content: {
            type: 'text',
            text: `Analyze this ${args.language} code for potential issues...`,
          },
        },
      ],
    }
  }

  throw new Error(`Unknown prompt: ${name}`)
})
```

### 动态工具发现

```typescript
// 运行时添加工具
let dynamicTools: Tool[] = []

server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [...staticTools, ...dynamicTools],
  }
})

// 通过通知更新工具
function addTool(tool: Tool): void {
  dynamicTools.push(tool)
  server.notification({
    method: 'notifications/tools/list_changed',
  })
}
```

## 错误处理

### 工具错误

```typescript
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  try {
    const result = await executeTool(request.params)
    return {
      content: [{ type: 'text', text: result }],
    }
  } catch (error) {
    return {
      content: [
        {
          type: 'text',
          text: `Error: ${error instanceof Error ? error.message : 'Unknown error'}`,
        },
      ],
      isError: true,
    }
  }
})
```

### 验证输入

```typescript
import { z } from 'zod'

const MyToolSchema = z.object({
  input: z.string().min(1),
  count: z.number().int().positive().optional(),
})

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params

  if (name === 'my_tool') {
    const parsed = MyToolSchema.safeParse(args)
    if (!parsed.success) {
      return {
        content: [
          { type: 'text', text: `Invalid input: ${parsed.error.message}` },
        ],
        isError: true,
      }
    }

    // 使用验证后的输入
    const result = await processTool(parsed.data)
    return { content: [{ type: 'text', text: result }] }
  }
})
```

## 测试 MCP 服务器

### 单元测试

```typescript
import { test, expect, describe } from 'bun:test'
import { handleToolCall } from './handler'

describe('my_tool', () => {
  test('processes input correctly', async () => {
    const result = await handleToolCall('my_tool', { input: 'test' })
    expect(result.content[0].text).toBe('expected output')
  })

  test('handles errors', async () => {
    const result = await handleToolCall('my_tool', {})
    expect(result.isError).toBe(true)
  })
})
```

### 集成测试

```typescript
import { test, expect } from 'bun:test'
import { Client } from '@modelcontextprotocol/sdk/client/index.js'
import { StdioClientTransport } from '@modelcontextprotocol/sdk/client/stdio.js'

test('server responds to tool list', async () => {
  const transport = new StdioClientTransport({
    command: 'node',
    args: ['./dist/index.js'],
  })

  const client = new Client({ name: 'test', version: '1.0' }, {})
  await client.connect(transport)

  const { tools } = await client.request(
    { method: 'tools/list' },
    ListToolsResultSchema
  )

  expect(tools.length).toBeGreaterThan(0)
  expect(tools.find(t => t.name === 'my_tool')).toBeDefined()

  await client.close()
})
```

## 打包和发布

### package.json

```json
{
  "name": "@my-org/mcp-server",
  "version": "1.0.0",
  "type": "module",
  "bin": {
    "mcp-server": "./dist/index.js"
  },
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.0"
  }
}
```

### 发布到 npm

```bash
# 构建
bun run build

# 发布
npm publish --access public
```

## 最佳实践

### 1. 清晰的工具描述

```typescript
{
  name: 'search_docs',
  description: `
Search the documentation for relevant information.

Returns up to 10 most relevant document sections.

Args:
  query: The search query (required)
  limit: Maximum number of results (optional, default 10)
  `.trim(),
  inputSchema: { /* ... */ },
}
```

### 2. 结构化输出

```typescript
// 返回 JSON 格式的结构化数据
function formatResult(data: unknown): string {
  return JSON.stringify(data, null, 2)
}

// 或 Markdown 格式
function formatAsMarkdown(data: SearchResult[]): string {
  return data.map(r => `## ${r.title}\n\n${r.content}`).join('\n\n---\n\n')
}
```

### 3. 超时处理

```typescript
async function withTimeout<T>(
  promise: Promise<T>,
  ms: number,
): Promise<T> {
  const timeout = new Promise<never>((_, reject) =>
    setTimeout(() => reject(new Error('Timeout')), ms),
  )
  return Promise.race([promise, timeout])
}

const result = await withTimeout(fetchData(), 30000)
```

### 4. 日志记录

```typescript
// 使用 stderr 输出日志（stdout 用于 MCP 通信）
console.error('[INFO] Tool called with args:', args)
console.error('[ERROR] Failed to process:', error)
```

## 关键文件

- [src/services/mcp/client.ts](src/services/mcp/client.ts) — MCP 客户端
- [src/services/mcp/types.ts](src/services/mcp/types.ts) — 类型定义
- [src/tools/MCPTool/MCPTool.ts](src/tools/MCPTool/MCPTool.ts) — MCP 工具

## 关联文档

- [06-MCP协议/MCP集成详解-MCP-Integration.md](../06-MCP协议/MCP集成详解-MCP-Integration.md)
- [06-MCP协议/MCP协议实现-MCP-Protocol-Implementation.md](../06-MCP协议/MCP协议实现-MCP-Protocol-Implementation.md)
- [11-插件系统/插件系统-Plugin-System.md](../11-插件系统/插件系统-Plugin-System.md)