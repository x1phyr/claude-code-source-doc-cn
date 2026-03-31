# Claude Code 项目架构总览

## 概述

Claude Code 是 Anthropic 官方的命令行工具，允许用户在终端中与 Claude 模型进行交互式对话，执行代码编写、文件操作、命令执行等任务。本项目是一个从 source map 还原的源码树，使用 Bun + TypeScript + Ink 构建。

## 技术栈

| 技术 | 用途 |
|------|------|
| **Bun** | JavaScript 运行时和包管理器 |
| **TypeScript** | 类型安全的 JavaScript 超集 |
| **ESM** | ES Module 模块系统 |
| **Ink** | React for CLI - 用 React 语法构建终端 UI |
| **React** | UI 组件框架 |
| **Zod** | 运行时类型校验库 |
| **Anthropic SDK** | Claude API 客户端 |

## 核心架构

### 入口点链路

```
src/bootstrap-entry.ts          # 最顶层入口，注入构建时常量
    └── src/entrypoints/cli.tsx # CLI 参数解析和模式分发
        └── src/main.tsx        # REPL 主渲染器 (800KB+)
            └── src/QueryEngine.ts  # 查询引擎，核心对话逻辑
```

### 核心模块关系

```
┌─────────────────────────────────────────────────────────────────┐
│                         用户输入                                 │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  processUserInput (utils/processUserInput/)                     │
│  - 处理斜杠命令                                                  │
│  - 处理附件、图片                                                │
│  - 构建 Message 对象                                             │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  QueryEngine / query() (QueryEngine.ts, query.ts)               │
│  - 管理会话状态                                                  │
│  - 调用 AutoCompact (上下文压缩)                                 │
│  - 调用 Claude API                                               │
│  - 处理工具调用                                                  │
│  - 生成响应消息                                                  │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  Tool System (tools/)                                           │
│  - BashTool: 执行 shell 命令                                    │
│  - FileReadTool/FileEditTool/FileWriteTool: 文件操作            │
│  - AgentTool: 启动子代理                                        │
│  - 其他 60+ 工具                                                 │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                         响应输出                                 │
│  - Ink 组件渲染到终端                                           │
│  - SDK 消息流输出                                               │
└─────────────────────────────────────────────────────────────────┘
```

### 数据流向

1. **用户输入** → `processUserInput()` 解析和处理
2. **消息构建** → 创建 `Message` 对象，处理附件
3. **上下文准备** → `getSystemContext()` 和 `getUserContext()` 构建系统提示
4. **查询执行** → `query()` 函数执行主循环
5. **API 调用** → `callModel()` 调用 Claude API
6. **工具执行** → `runTools()` 并行执行工具调用
7. **响应生成** → 生成 assistant 消息和工具结果
8. **UI 更新** → Ink 组件重新渲染

## 核心系统

### 1. 查询引擎 (QueryEngine)

`QueryEngine` 是会话管理的核心类，负责：
- 管理会话消息列表 (`mutableMessages`)
- 处理用户提交 (`submitMessage()`)
- 追踪使用量和成本
- 处理权限拒绝记录
- 支持中断操作

```typescript
// 核心配置结构
type QueryEngineConfig = {
  cwd: string                    // 工作目录
  tools: Tools                   // 可用工具列表
  commands: Command[]            // 可用命令列表
  mcpClients: MCPServerConnection[]  // MCP 客户端
  canUseTool: CanUseToolFn       // 权限检查函数
  initialMessages?: Message[]    // 初始消息
  thinkingConfig?: ThinkingConfig  // 思考配置
  maxTurns?: number              // 最大轮次
  maxBudgetUsd?: number          // 预算限制
}
```

### 2. 工具系统 (Tool System)

每个工具实现 `Tool` 接口，包含：
- `name`: 工具名称
- `inputSchema`: Zod 定义的输入 schema
- `call()`: 执行工具逻辑
- `checkPermissions()`: 权限检查
- `renderToolUseMessage()`: 渲染工具调用 UI
- `renderToolResultMessage()`: 渲染工具结果 UI

工具生命周期：
1. `isEnabled()` → 检查是否启用
2. `validateInput()` → 验证输入参数
3. `checkPermissions()` → 检查权限
4. `call()` → 执行工具

### 3. 命令系统 (Command System)

命令分为三种类型：
- `prompt`: 展开为发送给模型的文本 (skills, workflows)
- `local`: 本地执行，返回文本输出
- `local-jsx`: 渲染 Ink UI 组件

### 4. 权限系统 (Permission System)

权限模式 (`PermissionMode`):
- `default`: 默认模式，需要用户确认
- `acceptEdits`: 自动接受编辑操作
- `bypassPermissions`: 绕过所有权限检查
- `auto`: 自动模式，使用分类器判断

### 5. MCP 集成 (Model Context Protocol)

MCP 允许外部工具和资源集成：
- 工具发现和调用
- 资源访问
- OAuth 认证

### 6. 状态管理 (State Management)

使用 React Context 和 hooks 管理全局状态：
- `AppState`: 全局应用状态
- 消息列表管理
- 文件缓存
- 会话恢复

## 关键文件

### 入口和启动
- `src/bootstrap-entry.ts` - 入口点
- `src/bootstrapMacro.ts` - 构建时常量
- `src/entrypoints/cli.tsx` - CLI 分发
- `src/main.tsx` - REPL 主渲染器

### 核心逻辑
- `src/QueryEngine.ts` - 查询引擎
- `src/query.ts` - 查询主循环
- `src/Tool.ts` - 工具类型定义
- `src/tools.ts` - 工具注册
- `src/commands.ts` - 命令注册
- `src/context.ts` - 上下文构建

### 类型定义
- `src/types/message.ts` - 消息类型
- `src/types/permissions.ts` - 权限类型
- `src/types/command.ts` - 命令类型
- `src/types/tools.ts` - 工具类型

## 还原状态说明

本仓库是从 source map 还原的源码，存在以下限制：

1. **类型专用文件缺失** - 部分类型定义不完整
2. **构建时生成文件缺失** - 某些动态生成的代码不存在
3. **私有包包装层不可用** - 需要用 shim 替代
4. **原生绑定降级** - Native 模块使用 JS 回退实现

## 关联文档

- [02-目录结构详解](02-directory-structure.md)
- [03-启动流程](03-entry-and-bootstrap.md)
- [04-查询引擎](04-query-engine.md)
- [05-工具系统](05-tool-system.md)