# 查询引擎详解

## 概述

`QueryEngine` 是 Claude Code 的核心类，负责管理会话状态和处理用户消息。它将查询生命周期从 REPL 中提取出来，使 SDK 和交互模式可以共享逻辑。

## 核心类：QueryEngine

### 定义位置

`src/QueryEngine.ts`

### 配置结构

```typescript
type QueryEngineConfig = {
  // 基本信息
  cwd: string                    // 工作目录
  tools: Tools                   // 可用工具列表
  commands: Command[]            // 可用命令列表
  mcpClients: MCPServerConnection[]  // MCP 客户端
  agents: AgentDefinition[]      // 代理定义

  // 权限
  canUseTool: CanUseToolFn       // 权限检查函数
  getAppState: () => AppState    // 获取应用状态
  setAppState: (f: (prev: AppState) => AppState) => void  // 更新状态

  // 会话
  initialMessages?: Message[]    // 初始消息
  readFileCache: FileStateCache  // 文件读取缓存

  // 系统提示
  customSystemPrompt?: string    // 自定义系统提示
  appendSystemPrompt?: string    // 追加系统提示

  // 模型
  userSpecifiedModel?: string    // 用户指定模型
  fallbackModel?: string         // 备用模型
  thinkingConfig?: ThinkingConfig  // 思考配置

  // 限制
  maxTurns?: number              // 最大轮次
  maxBudgetUsd?: number          // 预算限制（美元）
  taskBudget?: { total: number } // 任务预算

  // 输出
  jsonSchema?: Record<string, unknown>  // JSON Schema 输出
  verbose?: boolean              // 详细模式

  // 流程控制
  abortController?: AbortController  // 中断控制
  replayUserMessages?: boolean       // 重放用户消息
  includePartialMessages?: boolean   // 包含部分消息
  handleElicitation?: ...            // MCP elicitation 处理

  // 状态
  orphanedPermission?: OrphanedPermission  // 孤立权限
}
```

### 核心属性

```typescript
class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]      // 可变消息列表
  private abortController: AbortController  // 中断控制
  private permissionDenials: SDKPermissionDenial[]  // 权限拒绝记录
  private totalUsage: NonNullableUsage     // 总使用量
  private readFileState: FileStateCache    // 文件读取状态
  private discoveredSkillNames: Set<string>  // 发现的技能
  private loadedNestedMemoryPaths: Set<string>  // 加载的嵌套记忆路径
}
```

### 核心方法

#### submitMessage()

提交用户消息并生成响应：

```typescript
async *submitMessage(
  prompt: string | ContentBlockParam[],
  options?: { uuid?: string; isMeta?: boolean }
): AsyncGenerator<SDKMessage, void, unknown>
```

**生命周期**：

1. **初始化**
   - 清理上一轮状态
   - 设置工作目录
   - 创建权限包装器

2. **准备上下文**
   - 获取系统提示
   - 获取用户上下文（CLAUDE.md 等）
   - 构建完整系统提示

3. **处理用户输入**
   - 调用 `processUserInput()` 处理斜杠命令
   - 处理附件和图片
   - 构建消息对象

4. **持久化**
   - 记录会话日志
   - 写入 transcript

5. **查询循环**
   - 调用 `query()` 执行主循环
   - 处理各种消息类型
   - 生成 SDK 消息

6. **结果生成**
   - 计算成本和使用量
   - 生成最终结果消息

## query() 函数

### 定义位置

`src/query.ts`

### 核心循环

```typescript
async function* query(params: QueryParams): AsyncGenerator<...> {
  while (true) {
    // 1. 准备消息
    let messagesForQuery = [...getMessagesAfterCompactBoundary(messages)]

    // 2. 应用上下文压缩
    // - Snip compact
    // - Microcompact
    // - Context collapse
    // - Auto compact

    // 3. 调用 API
    for await (const message of deps.callModel({...})) {
      yield message

      if (message.type === 'assistant') {
        // 收集工具调用
        if (hasToolUse) {
          needsFollowUp = true
        }
      }
    }

    // 4. 执行工具
    if (needsFollowUp) {
      for await (const update of runTools(...)) {
        yield update.message
      }

      // 递归继续下一轮
      state = {...}
      continue
    }

    // 5. 完成
    return { reason: 'completed' }
  }
}
```

### 状态机

```
┌─────────────┐
│   开始      │
└─────────────┘
      │
      ▼
┌─────────────┐     ┌──────────────┐
│  准备消息    │────►│  上下文压缩   │
└─────────────┘     └──────────────┘
      │                    │
      ▼                    │
┌─────────────┐            │
│  调用 API   │◄───────────┘
└─────────────┘
      │
      ▼
┌─────────────┐
│ 有工具调用？ │
└─────────────┘
      │
  是  │  否
      │   │
      ▼   ▼
┌─────────────┐  ┌──────────────┐
│  执行工具    │  │   结束       │
└─────────────┘  └──────────────┘
      │
      │ (继续循环)
      └──────────────────►
```

### 继续条件（Continue Reasons）

| reason | 说明 |
|--------|------|
| `next_turn` | 正常下一轮 |
| `max_output_tokens_recovery` | 输出 token 限制恢复 |
| `max_output_tokens_escalate` | 输出 token 限制升级 |
| `stop_hook_blocking` | 停止钩子阻塞 |
| `collapse_drain_retry` | 上下文折叠重试 |
| `reactive_compact_retry` | 响应式压缩重试 |
| `token_budget_continuation` | Token 预算继续 |

## 消息类型

### 内部消息类型

```typescript
type Message =
  | UserMessage           // 用户消息
  | AssistantMessage      // 助手消息
  | ProgressMessage       // 进度消息
  | AttachmentMessage     // 附件消息
  | SystemMessage         // 系统消息
  | StreamEvent           // 流事件
  | TombstoneMessage      // 墓碑消息（删除标记）
  | ToolUseSummaryMessage // 工具调用摘要
```

### SDK 消息类型

```typescript
type SDKMessage =
  | { type: 'user', ... }
  | { type: 'assistant', ... }
  | { type: 'system', subtype: 'compact_boundary' | 'api_retry', ... }
  | { type: 'result', subtype: 'success' | 'error_*', ... }
  | { type: 'stream_event', ... }
  | { type: 'tool_use_summary', ... }
```

## 上下文压缩

### 压缩策略

1. **Snip Compact** (`HISTORY_SNIP`)
   - 快速删除旧消息
   - 保留关键上下文

2. **Microcompact**
   - 合并相邻的 assistant/user 消息对
   - 减少 token 数量

3. **Context Collapse** (`CONTEXT_COLLAPSE`)
   - 智能折叠上下文
   - 保留重要信息

4. **Auto Compact**
   - 自动触发压缩
   - 使用 Claude 生成摘要

### 压缩触发条件

- Token 数量超过阈值
- 用户请求 `/compact`
- API 返回 `prompt_too_long` 错误

## 会话持久化

### 存储位置

- `~/.claude/sessions/` - 会话日志
- `~/.claude/storage/` - 会话存储

### 持久化时机

- 用户消息发送后立即写入
- 助手消息流式完成后写入
- 压缩边界时刷新

## 错误处理

### 可恢复错误

| 错误 | 恢复策略 |
|------|----------|
| `prompt_too_long` | 自动压缩上下文 |
| `max_output_tokens` | 继续生成 |
| `model_overloaded` | 切换备用模型 |
| `rate_limit` | 等待重试 |

### 不可恢复错误

- 认证失败
- 配额耗尽
- 严重内部错误

## 关键文件

- `src/QueryEngine.ts` - 查询引擎类
- `src/query.ts` - 查询主循环
- `src/utils/messages.ts` - 消息工具函数
- `src/services/compact/` - 压缩服务

## 关联文档

- [01-架构总览](01-architecture-overview.md)
- [05-工具系统](05-tool-system.md)
- [09-状态管理](09-state-management.md)