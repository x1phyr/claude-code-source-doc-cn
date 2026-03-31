# 服务层详解

## 概述

服务层是 Claude Code 的核心业务逻辑层，位于 `src/services/` 目录下。每个服务负责特定的功能域，提供模块化、可复用的能力。

## 服务目录结构

```
src/services/
├── AgentSummary/          # 代理摘要
├── MagicDocs/             # 魔法文档
├── PromptSuggestion/      # 提示建议
├── SessionMemory/         # 会话记忆
├── analytics/             # 分析与追踪
├── api/                   # API 客户端
├── autoDream/             # 自动梦境
├── compact/               # 上下文压缩
├── contextCollapse/       # 上下文折叠
├── extractMemories/       # 记忆提取
├── lsp/                   # LSP 集成
├── mcp/                   # MCP 集成
├── oauth/                 # OAuth 认证
├── skillSearch/           # 技能搜索
└── ...                    # 更多服务
```

## API 服务

### 目录结构

[src/services/api/](src/services/api/)

```
api/
├── adminRequests.ts       # 管理请求
├── bootstrap.ts           # 引导 API
├── claude.ts              # Claude API 核心调用
├── client.ts              # HTTP 客户端
├── dumpPrompts.ts         # 提示转储
├── emptyUsage.ts          # 空使用量
├── errorUtils.ts          # 错误工具
├── errors.ts              # 错误类型
├── filesApi.ts            # 文件 API
├── firstTokenDate.ts      # 首次 Token 日期
├── grove.ts               # Grove API
├── logging.ts             # 日志
├── metricsOptOut.ts       # 指标退出
├── overageCreditGrant.ts  # 超额信用授权
├── promptCacheBreakDetection.ts  # 提示缓存检测
├── referral.ts            # 推荐
├── sessionIngress.ts      # 会话入口
├── ultrareviewQuota.ts    # 超审查配额
├── usage.ts               # 使用量
└── withRetry.ts           # 重试逻辑
```

### claude.ts

核心 API 调用模块：

```typescript
// 调用 Claude API
export async function* callClaudeAPI(
  messages: Message[],
  options: ClaudeAPIOptions
): AsyncGenerator<APIResponse>

// API 配置
type ClaudeAPIOptions = {
  model: string
  tools: Tools
  systemPrompt: string
  maxTokens: number
  temperature?: number
  thinking?: ThinkingConfig
}
```

### client.ts

HTTP 客户端封装：

```typescript
// 创建请求
export async function apiRequest<T>(
  endpoint: string,
  options: RequestOptions
): Promise<T>

// 带重试的请求
export async function apiRequestWithRetry<T>(
  endpoint: string,
  options: RequestOptions
): Promise<T>
```

### errors.ts

API 错误类型定义：

```typescript
export type APIError =
  | { type: 'authentication_error'; message: string }
  | { type: 'rate_limit'; retryAfter?: number }
  | { type: 'model_overloaded'; message: string }
  | { type: 'prompt_too_long'; message: string }
  | { type: 'max_output_tokens'; message: string }
  | { type: 'insufficient_quota'; message: string }
  | { type: 'unknown'; message: string }
```

## Compact 服务

### 目录结构

[src/services/compact/](src/services/compact/)

```
compact/
├── apiMicrocompact.ts     # API 微压缩
├── autoCompact.ts         # 自动压缩
├── cachedMCConfig.ts      # 缓存 MC 配置
├── compact.ts             # 主压缩逻辑
├── compactWarningHook.ts  # 压缩警告 Hook
├── compactWarningState.ts # 压缩警告状态
├── grouping.ts            # 消息分组
├── microCompact.ts        # 微压缩
├── postCompactCleanup.ts  # 压缩后清理
├── prompt.ts              # 压缩提示
├── reactiveCompact.ts     # 响应式压缩
├── sessionMemoryCompact.ts # 会话记忆压缩
├── snipCompact.ts         # 剪切压缩
├── snipProjection.ts      # 剪切投影
└── timeBasedMCConfig.ts   # 基于时间的 MC 配置
```

### 压缩策略

#### Snip Compact

快速删除旧消息：

```typescript
export function snipCompact(
  messages: Message[],
  targetTokens: number
): Message[]
```

#### Microcompact

合并相邻消息对：

```typescript
export function microCompact(
  messages: Message[],
  options: MicroCompactOptions
): Promise<Message[]>
```

#### Context Collapse

智能折叠上下文：

```typescript
export function contextCollapse(
  messages: Message[],
  options: CollapseOptions
): Promise<Message[]>
```

#### Auto Compact

自动触发压缩：

```typescript
export function checkAutoCompact(
  messages: Message[],
  tokenCount: number
): boolean

export async function runAutoCompact(
  messages: Message[]
): Promise<Message[]>
```

### 触发条件

| 触发方式 | 说明 |
|----------|------|
| Token 阈值 | 超过配置的 token 限制 |
| `/compact` 命令 | 用户手动触发 |
| `prompt_too_long` 错误 | API 返回错误后自动压缩 |
| 响应式压缩 | 检测到上下文压力 |

## MCP 服务

### 目录结构

[src/services/mcp/](src/services/mcp/)

```
mcp/
├── InProcessTransport.ts   # 进程内传输
├── SdkControlTransport.ts  # SDK 控制传输
├── auth.ts                 # 认证
├── channelAllowlist.ts     # 渠道白名单
├── channelNotification.ts  # 渠道通知
├── channelPermissions.ts   # 渠道权限
├── claudeai.ts             # Claude.ai 集成
├── client.ts               # MCP 客户端
├── config.ts               # 配置管理
├── elicitationHandler.ts   # Elicitation 处理
├── envExpansion.ts         # 环境变量展开
├── headersHelper.ts        # 请求头辅助
├── mcpStringUtils.ts       # 字符串工具
├── normalization.ts        # 名称规范化
├── oauthPort.ts            # OAuth 端口
├── officialRegistry.ts     # 官方注册表
├── types.ts                # 类型定义
├── useManageMCPConnections.ts  # 连接管理 Hook
├── utils.ts                # 工具函数
├── vscodeSdkMcp.ts         # VSCode SDK MCP
├── xaa.ts                  # 跨应用访问
└── xaaIdpLogin.ts          # XAA IdP 登录
```

详见 [08-mcp-integration.md](08-mcp-integration.md)。

## LSP 服务

### 目录结构

[src/services/lsp/](src/services/lsp/)

```
lsp/
├── LSPClient.ts            # LSP 客户端
├── LSPDiagnosticRegistry.ts # 诊断注册表
├── LSPServerInstance.ts    # 服务器实例
├── LSPServerManager.ts     # 服务器管理器
├── config.ts               # 配置
├── manager.ts              # 管理器
├── passiveFeedback.ts      # 被动反馈
└── types.ts                # 类型定义
```

### LSPClient

```typescript
export class LSPClient {
  // 启动 LSP 服务器
  async startServer(languageId: string): Promise<void>

  // 发送请求
  async request<T>(method: string, params: unknown): Promise<T>

  // 获取定义
  async getDefinition(uri: string, position: Position): Promise<Location[]>

  // 获取引用
  async getReferences(uri: string, position: Position): Promise<Location[]>

  // 获取悬停信息
  async getHover(uri: string, position: Position): Promise<Hover>
}
```

### LSPServerManager

```typescript
export class LSPServerManager {
  // 获取或创建服务器实例
  getOrCreate(languageId: string): LSPServerInstance

  // 关闭所有服务器
  async closeAll(): Promise<void>
}
```

## Analytics 服务

### 目录结构

[src/services/analytics/](src/services/analytics/)

```
analytics/
├── config.ts               # 配置
├── datadog.ts              # Datadog 集成
├── firstPartyEventLogger.ts # 一方事件日志
├── firstPartyEventLoggingExporter.ts # 导出器
├── growthbook.ts           # Feature flags
├── index.ts                # 主入口
├── metadata.ts             # 元数据
├── sink.ts                 # 事件接收器
└── sinkKillswitch.ts       # 熔断开关
```

### 事件追踪

```typescript
// 记录事件
export function logEvent(
  eventName: string,
  metadata?: Record<string, unknown>
): void

// 设置用户属性
export function setUserProperty(key: string, value: unknown): void
```

### GrowthBook

```typescript
// 获取 Feature Flag
export function getFeatureValue<T>(
  featureKey: string,
  defaultValue: T
): T

// 缓存版本（可能过期）
export function getFeatureValue_CACHED_MAY_BE_STALE<T>(
  featureKey: string,
  defaultValue: T
): T
```

## SessionMemory 服务

### 目录结构

[src/services/SessionMemory/](src/services/SessionMemory/)

```
SessionMemory/
├── prompts.ts              # 提示模板
├── sessionMemory.ts        # 会话记忆主模块
└── sessionMemoryUtils.ts   # 工具函数
```

### 会话记忆

```typescript
export class SessionMemory {
  // 加载会话记忆
  async load(sessionId: string): Promise<Memory[]>

  // 保存会话记忆
  async save(sessionId: string, memories: Memory[]): Promise<void>

  // 提取新记忆
  async extract(messages: Message[]): Promise<Memory[]>
}
```

## PromptSuggestion 服务

### 目录结构

[src/services/PromptSuggestion/](src/services/PromptSuggestion/)

```
PromptSuggestion/
├── promptSuggestion.ts     # 提示建议
└── speculation.ts          # 预测执行
```

### 提示建议

```typescript
// 生成提示建议
export function generatePromptSuggestion(
  context: PromptContext
): Promise<PromptSuggestion>

// 是否启用提示建议
export function shouldEnablePromptSuggestion(): boolean
```

### Speculation

```typescript
// 启动预测执行
export function startSpeculation(
  prompt: string
): SpeculationHandle

// 检查预测结果
export function checkSpeculation(
  handle: SpeculationHandle
): SpeculationResult | null
```

## 其他服务

### AgentSummary

[src/services/AgentSummary/agentSummary.ts](src/services/AgentSummary/agentSummary.ts)

代理任务摘要生成。

### MagicDocs

[src/services/MagicDocs/](src/services/MagicDocs/)

文档处理和增强。

### autoDream

[src/services/autoDream/](src/services/autoDream/)

自动梦境（记忆整合）。

### extractMemories

[src/services/extractMemories/](src/services/extractMemories/)

从会话中提取记忆。

### contextCollapse

[src/services/contextCollapse/](src/services/contextCollapse/)

上下文折叠操作。

### skillSearch

[src/services/skillSearch/](src/services/skillSearch/)

技能搜索和索引。

### oauth

[src/services/oauth/](src/services/oauth/)

OAuth 认证流程。

## 服务依赖关系

```
┌─────────────────────────────────────────────────────────────┐
│                        CLI / REPL                           │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                       Query Engine                          │
└─────────────────────────────────────────────────────────────┘
        │           │           │           │
        ▼           ▼           ▼           ▼
┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐
│    API    │ │   MCP     │ │  Compact  │ │   LSP     │
│  Service  │ │  Service  │ │  Service  │ │  Service  │
└───────────┘ └───────────┘ └───────────┘ └───────────┘
        │           │           │           │
        ▼           ▼           ▼           ▼
┌─────────────────────────────────────────────────────────────┐
│                      Analytics                              │
└─────────────────────────────────────────────────────────────┘
```

## 关键文件

- [src/services/api/claude.ts](src/services/api/claude.ts) — API 核心调用
- [src/services/compact/compact.ts](src/services/compact/compact.ts) — 压缩主逻辑
- [src/services/mcp/client.ts](src/services/mcp/client.ts) — MCP 客户端
- [src/services/lsp/manager.ts](src/services/lsp/manager.ts) — LSP 管理器
- [src/services/analytics/index.ts](src/services/analytics/index.ts) — 分析入口

## 关联文档

- [04-查询引擎](04-query-engine.md)
- [08-MCP集成](08-mcp-integration.md)
- [07-权限系统](07-permission-system.md)