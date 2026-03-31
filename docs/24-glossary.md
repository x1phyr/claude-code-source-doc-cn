# 术语表

## 概述

本文档定义 Claude Code 项目中使用的专用术语和缩写。

## 核心概念

### REPL

**Read-Eval-Print Loop**：交互式命令行界面，读取用户输入、执行、打印结果，然后循环。

### Tool

**工具**：Claude 可以执行的操作，如读取文件、运行命令、搜索代码等。

### Command

**命令**：斜杠命令（如 `/help`、`/config`），可以是提示展开、本地执行或 UI 渲染。

### Skill

**技能**：一种特殊类型的命令，提供专门的能力。技能定义在 SKILL.md 文件中。

### MCP

**Model Context Protocol**：开放协议，允许 Claude 连接外部工具和资源。

## 架构术语

### QueryEngine

**查询引擎**：核心类，管理会话状态和处理用户消息。

### AppState

**应用状态**：全局状态对象，包含所有运行时数据。

### Store

**存储**：状态管理模式，提供 getState/setState/subscribe 方法。

### Compact

**压缩**：减少上下文 token 数量的过程，包括删除旧消息和生成摘要。

### Speculation

**预测执行**：在用户确认前预先执行可能的操作，优化响应速度。

## 权限术语

### PermissionMode

**权限模式**：控制操作许可的行为模式。

| 模式 | 说明 |
|------|------|
| default | 敏感操作需要确认 |
| acceptEdits | 编辑操作自动接受 |
| bypassPermissions | 所有操作自动允许 |
| dontAsk | 所有操作自动拒绝 |
| plan | 规划模式权限 |
| auto | 分类器自动决定 |

### PermissionRule

**权限规则**：定义允许、拒绝或询问特定操作。

### YoloClassifier

**自动分类器**：在 auto 模式下自动决定是否允许操作。

### Sandbox

**沙箱**：隔离环境执行命令，提供额外安全层。

## MCP 术语

### Transport

**传输**：MCP 服务器连接方式。

| 类型 | 说明 |
|------|------|
| stdio | 标准输入输出 |
| sse | Server-Sent Events |
| http | HTTP 请求 |
| ws | WebSocket |
| sdk | 内嵌 SDK |

### Elicitation

**请求**：MCP 服务器向用户请求信息或确认。

### Resource

**资源**：MCP 服务器提供的可访问资源，如文件、数据库记录等。

## 组件术语

### Ink

**Ink 框架**：React for CLI，用于构建终端用户界面。

### Hook

**钩子**：有两种含义：
1. React Hooks：函数组件状态管理
2. CLI Hooks：事件钩子，在特定事件点执行自定义逻辑

### Component

**组件**：React 组件，渲染 UI 部分。

### Provider

**提供者**：React Context 提供者，提供上下文给子组件。

## 消息术语

### Message

**消息**：会话中的单条记录。

| 类型 | 说明 |
|------|------|
| UserMessage | 用户消息 |
| AssistantMessage | 助手消息 |
| SystemMessage | 系统消息 |
| ProgressMessage | 进度消息 |
| ToolUseMessage | 工具调用消息 |

### Transcript

**会话记录**：完整会话的消息列表。

### Context

**上下文**：当前会话的所有信息，包括消息、状态等。

## 文件术语

### CLAUDE.md

项目级指令文件，提供项目上下文和开发指南。

### SKILL.md

技能定义文件，包含技能名称、描述和内容。

### MEMORY.md

记忆索引文件，列出所有记忆。

### settings.json

配置文件，定义权限、MCP 服务器、Hooks 等。

## 代理术语

### Agent

**代理**：独立执行的子任务，有自己的工具集和上下文。

### Subagent

**子代理**：Agent 工具创建的代理。

### Teammate

**队友**：多代理系统中的协作代理。

### Swarm

**群体**：多个代理协作执行任务。

### Coordinator

**协调者**：管理多个代理的协调器。

## 任务术语

### Task

**任务**：后台执行的操作单元。

### Foreground Task

**前台任务**：当前显示的任务。

### Background Task

**后台任务**：在后台执行的任务。

### Worktree

**工作树**：Git 工作树，提供隔离的工作目录。

## API 术语

### Token

**Token**：文本的基本单位，用于计算上下文大小和成本。

### Thinking

**思考模式**：模型的扩展推理能力。

### Streaming

**流式传输**：逐步接收模型输出，而非等待完整响应。

### Cache

**缓存**：存储常用数据以加速后续请求。

## 缩写

| 缩写 | 全称 |
|------|------|
| CLI | Command Line Interface |
| API | Application Programming Interface |
| JSON | JavaScript Object Notation |
| AST | Abstract Syntax Tree |
| LSP | Language Server Protocol |
| SDK | Software Development Kit |
| OAuth | Open Authorization |
| SSE | Server-Sent Events |
| WS | WebSocket |

## 关联文档

- [01-架构总览](01-architecture-overview.md)
- [21-类型参考](21-types-reference.md)