# 配置参考

## 概述

本文档详细说明 Claude Code 的配置文件格式和选项。

## 配置文件位置

| 作用域 | 路径 | 优先级 |
|--------|------|--------|
| User | `~/.claude/settings.json` | 低 |
| Project | `.claude/settings.json` | 中 |
| Local | `.claude/settings.local.json` | 高 |

## 完整配置示例

```json
{
  "model": "claude-sonnet-4-6",

  "permissions": {
    "mode": "default",
    "allow": ["Read", "Edit", "Write"],
    "deny": ["Bash(rm -rf /*)"],
    "ask": ["Bash(git push *)"]
  },

  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path"]
    },
    "github": {
      "type": "http",
      "url": "https://api.github.com/mcp",
      "headers": {
        "Authorization": "Bearer ${GITHUB_TOKEN}"
      }
    }
  },

  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": ["echo 'Before Bash'"]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": ["prettier --write $FILE_PATH"]
      }
    ]
  },

  "theme": "dark",

  "outputStyle": "default",

  "thinkingEnabled": true,

  "effort": "medium"
}
```

## 权限配置

### 模式

```json
{
  "permissions": {
    "mode": "default" | "acceptEdits" | "bypassPermissions" | "plan" | "auto"
  }
}
```

| 模式 | 说明 |
|------|------|
| `default` | 敏感操作需要用户确认 |
| `acceptEdits` | 文件编辑自动接受 |
| `bypassPermissions` | 所有操作自动允许（危险） |
| `plan` | 规划模式权限规则 |
| `auto` | 使用分类器自动决定 |

### 规则

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Edit",
      "Bash(npm run *)"
    ],
    "deny": [
      "Bash(rm -rf /*)",
      "Bash(sudo *)"
    ],
    "ask": [
      "Bash(git push *)",
      "Write"
    ]
  }
}
```

### 规则语法

```
ToolName                    # 匹配工具
ToolName(pattern)           # 匹配工具 + 输入模式
mcp__server__tool           # 匹配 MCP 工具
```

## MCP 服务器配置

### stdio 类型

```json
{
  "mcpServers": {
    "my-server": {
      "command": "npx",
      "args": ["-y", "@scope/server-package"],
      "env": {
        "API_KEY": "${MY_API_KEY}"
      }
    }
  }
}
```

### SSE 类型

```json
{
  "mcpServers": {
    "my-sse-server": {
      "type": "sse",
      "url": "http://localhost:3000/sse",
      "headers": {
        "Authorization": "Bearer ${TOKEN}"
      }
    }
  }
}
```

### HTTP 类型

```json
{
  "mcpServers": {
    "my-http-server": {
      "type": "http",
      "url": "https://api.example.com/mcp",
      "headers": {
        "X-API-Key": "${API_KEY}"
      }
    }
  }
}
```

### WebSocket 类型

```json
{
  "mcpServers": {
    "my-ws-server": {
      "type": "ws",
      "url": "wss://ws.example.com/mcp"
    }
  }
}
```

### OAuth 配置

```json
{
  "mcpServers": {
    "oauth-server": {
      "type": "sse",
      "url": "https://api.example.com/sse",
      "oauth": {
        "clientId": "my-client-id",
        "callbackPort": 8080,
        "authServerMetadataUrl": "https://auth.example.com/.well-known/oauth-authorization-server"
      }
    }
  }
}
```

## Hooks 配置

### PreToolUse

工具执行前触发：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": ["security-check.sh"]
      }
    ]
  }
}
```

### PostToolUse

工具执行后触发：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": ["prettier --write $FILE_PATH"]
      }
    ]
  }
}
```

### Stop

会话停止时触发：

```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": "",
        "hooks": ["cleanup.sh"]
      }
    ]
  }
}
```

### 环境变量

Hook 执行时的环境变量：

| 变量 | 说明 |
|------|------|
| `CLAUDE_TOOL_NAME` | 工具名称 |
| `CLAUDE_TOOL_INPUT` | 工具输入（JSON） |
| `CLAUDE_TOOL_OUTPUT` | 工具输出（JSON） |
| `CLAUDE_SESSION_ID` | 会话 ID |
| `CLAUDE_CWD` | 工作目录 |

## 模型配置

### 指定模型

```json
{
  "model": "claude-sonnet-4-6"
}
```

### 模型别名

| 别名 | 完整 ID |
|------|---------|
| `opus` | `claude-opus-4-6-20250801` |
| `sonnet` | `claude-sonnet-4-6-20250801` |
| `haiku` | `claude-haiku-4-5-20251001` |

## 主题配置

```json
{
  "theme": "dark" | "light" | "system"
}
```

## 输出风格配置

```json
{
  "outputStyle": "default" | "compact" | "verbose"
}
```

| 风格 | 说明 |
|------|------|
| `default` | 默认输出 |
| `compact` | 紧凑输出 |
| `verbose` | 详细输出 |

## 思考模式配置

```json
{
  "thinkingEnabled": true
}
```

## Effort 配置

```json
{
  "effort": "low" | "medium" | "high"
}
```

| 级别 | 说明 |
|------|------|
| `low` | 快速响应，较低深度 |
| `medium` | 平衡响应 |
| `high` | 深度分析，较慢 |

## 禁用工具

```json
{
  "disabledTools": ["Bash", "WebSearch"]
}
```

## 环境变量

配置中支持环境变量展开：

```json
{
  "mcpServers": {
    "api": {
      "headers": {
        "Authorization": "Bearer ${API_TOKEN}"
      }
    }
  }
}
```

## 配置来源优先级

1. CLI 参数（最高）
2. 本地设置 (`.claude/settings.local.json`)
3. 项目设置 (`.claude/settings.json`)
4. 用户设置 (`~/.claude/settings.json`)
5. 默认值（最低）

## 关键文件

- [src/utils/settings/types.ts](src/utils/settings/types.ts) — 设置类型
- [src/utils/settings/constants.ts](src/utils/settings/constants.ts) — 设置常量
- [src/utils/settings/settings.ts](src/utils/settings/settings.ts) — 设置管理

## 关联文档

- [07-权限系统](07-permission-system.md)
- [08-MCP集成](08-mcp-integration.md)
- [18-Hook系统](18-hooks-system.md)