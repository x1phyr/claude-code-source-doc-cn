# 工具函数清单

## 概述

`src/utils/` 目录包含 100+ 个工具函数模块，提供各种通用功能。

## 目录结构

```
src/utils/
├── bash/                  # Shell 解析与执行
├── permissions/           # 权限处理
├── model/                 # 模型配置
├── git/                   # Git 操作
├── filesystem/            # 文件系统
├── settings/              # 设置管理
├── hooks/                 # Hook 工具
├── task/                  # 任务工具
├── shell/                 # Shell 工具
├── sandbox/               # 沙箱工具
└── ...                    # 更多工具
```

## Bash 工具

### 目录结构

[src/utils/bash/](src/utils/bash/)

```
bash/
├── ast.ts                 # AST 解析
├── commands.ts            # 命令解析
├── escaping.ts            # 转义处理
├── operators.ts           # 操作符处理
└── quotes.ts              # 引号处理
```

### 核心函数

```typescript
// 解析 Shell 命令 AST
export function parseForSecurity(command: string): BashAST

// 分割命令
export function splitCommand_DEPRECATED(command: string): string[]
export function splitCommandWithOperators(command: string): string[]

// 解析管道
export function parsePipeline(command: string): Pipeline
```

## 权限工具

### 目录结构

[src/utils/permissions/](src/utils/permissions/)

```
permissions/
├── permissions.ts         # 权限核心逻辑
├── PermissionMode.ts      # 权限模式
├── PermissionResult.ts    # 权限结果
├── permissionSetup.ts     # 权限设置
├── shellRuleMatching.ts   # Shell 规则匹配
├── filesystem.ts          # 文件系统权限
└── denialTracking.ts      # 拒绝追踪
```

### 核心函数

```typescript
// 获取工具的拒绝规则
export function getDenyRuleForTool(
  context: ToolPermissionContext,
  tool: Tool
): PermissionRule | undefined

// 检查权限
export function checkToolPermission(
  tool: Tool,
  input: unknown,
  context: ToolPermissionContext
): Promise<PermissionResult>

// 匹配通配符模式
export function matchWildcardPattern(
  pattern: string,
  value: string
): boolean
```

## 模型工具

### 目录结构

[src/utils/model/](src/utils/model/)

```
model/
├── model.ts               # 模型配置
├── agent.ts               # 代理模型
├── providers.ts           # 提供商
└── canonicalNames.ts      # 规范名称
```

### 核心函数

```typescript
// 获取规范名称
export function getCanonicalName(model: string): string

// 获取主循环模型
export function getMainLoopModel(): ModelSetting

// 检查是否支持思考模式
export function supportsThinking(model: string): boolean

// 获取模型提供商
export function getProvider(model: string): Provider
```

## Git 工具

### 目录结构

[src/utils/git/](src/utils/git/)

```
git/
├── operations.ts          # Git 操作
├── status.ts              # Git 状态
├── branch.ts              # 分支操作
├── commit.ts              # 提交操作
└── diff.ts                # 差异处理
```

### 核心函数

```typescript
// 获取 Git 状态
export async function getGitStatus(cwd: string): Promise<GitStatus>

// 获取当前分支
export async function getCurrentBranch(cwd: string): Promise<string>

// 获取差异
export async function getDiff(cwd: string): Promise<string>

// 创建提交
export async function createCommit(
  cwd: string,
  message: string
): Promise<void>
```

## 文件系统工具

### 目录结构

[src/utils/filesystem/](src/utils/filesystem/)

```
filesystem/
├── operations.ts          # 文件操作
├── watching.ts            # 文件监视
├── paths.ts               # 路径处理
└── encoding.ts            # 编码检测
```

### 核心函数

```typescript
// 读取文件
export async function readFile(path: string): Promise<string>

// 写入文件
export async function writeFile(
  path: string,
  content: string
): Promise<void>

// 检测文件编码
export function detectFileEncoding(buffer: Buffer): string

// 获取文件修改时间
export async function getFileModificationTime(
  path: string
): Promise<number>
```

## 设置工具

### 目录结构

[src/utils/settings/](src/utils/settings/)

```
settings/
├── settings.ts            # 设置主模块
├── types.ts               # 设置类型
├── constants.ts           # 设置常量
├── applySettingsChange.ts # 应用变更
└── mergeSettings.ts       # 合并设置
```

### 核心函数

```typescript
// 获取初始设置
export function getInitialSettings(): SettingsJson

// 应用设置变更
export function applySettingsChange(
  source: SettingSource,
  setState: (updater: (prev: AppState) => AppState) => void
): void

// 合并设置
export function mergeSettings(
  base: SettingsJson,
  override: Partial<SettingsJson>
): SettingsJson
```

## Hook 工具

### 目录结构

[src/utils/hooks/](src/utils/hooks/)

```
hooks/
├── postSamplingHooks.ts   # 采样后 Hooks
├── sessionHooks.ts        # 会话 Hooks
├── hookRunner.ts          # Hook 运行器
└── hookTypes.ts           # Hook 类型
```

### 核心函数

```typescript
// 运行 Hook
export async function runHook(
  hook: Hook,
  context: HookContext
): Promise<HookResult>

// 注册 Hook
export function registerHook(
  name: string,
  hook: Hook
): void

// 获取会话 Hooks
export function getSessionHooks(): Map<string, Hook>
```

## 任务工具

### 目录结构

[src/utils/task/](src/utils/task/)

```
task/
├── TaskOutput.ts          # 任务输出
├── diskOutput.ts          # 磁盘输出
├── taskState.ts           # 任务状态
└── taskUtils.ts           # 任务工具
```

### 核心函数

```typescript
// 获取任务输出路径
export function getTaskOutputPath(taskId: string): string

// 创建任务输出
export function createTaskOutput(
  taskId: string,
  output: unknown
): Promise<void>

// 获取任务状态
export function getTaskState(taskId: string): TaskState
```

## Shell 工具

### 目录结构

[src/utils/shell/](src/utils/shell/)

```
shell/
├── Shell.ts               # Shell 执行
├── ShellCommand.ts        # 命令执行
├── shellToolUtils.ts      # 工具函数
└── detection.ts           # Shell 检测
```

### 核心函数

```typescript
// 执行命令
export async function exec(
  command: string,
  options: ExecOptions
): Promise<ExecResult>

// 检测 Shell
export function detectShell(): ShellType

// 解析命令输出
export function parseCommandOutput(output: string): string[]
```

## 沙箱工具

### 目录结构

[src/utils/sandbox/](src/utils/sandbox/)

```
sandbox/
├── sandbox-adapter.ts     # 沙箱适配器
├── sandbox-exec.ts        # 沙箱执行
└── sandbox-config.ts      # 沙箱配置
```

### 核心函数

```typescript
// 检查是否使用沙箱
export function shouldUseSandbox(command: string): boolean

// 在沙箱中执行
export async function execInSandbox(
  command: string,
  options: SandboxOptions
): Promise<ExecResult>
```

## 其他工具

### 日志

```typescript
// src/utils/log.ts
export function logError(error: Error): void
export function logForDebugging(message: string): void
```

### 错误处理

```typescript
// src/utils/errors.ts
export function toError(unknown: unknown): Error
export function isENOENT(error: unknown): boolean
export function getErrnoCode(error: unknown): string | undefined
```

### 路径处理

```typescript
// src/utils/path.ts
export function expandPath(path: string): string
export function isAbsolute(path: string): boolean
export function resolvePath(base: string, relative: string): string
```

### Token 估算

```typescript
// src/services/tokenEstimation.ts
export function roughTokenCountEstimation(text: string): number
export function countTokensWithAPI(text: string): Promise<number>
```

### 消息处理

```typescript
// src/utils/messages.ts
export function createUserMessage(content: string): UserMessage
export function extractTextContent(message: Message): string
export function normalizeMessages(messages: Message[]): Message[]
```

### UUID 生成

```typescript
// src/utils/uuid.ts
export function createId(): string
export function createAgentId(): AgentId
```

### 休眠

```typescript
// src/utils/sleep.ts
export function sleep(ms: number): Promise<void>
```

## 设计原则

### 单一职责

每个工具模块只负责一个功能域。

### 纯函数优先

尽可能使用纯函数，避免副作用。

### 类型安全

所有函数都有明确的 TypeScript 类型签名。

### 错误处理

所有异步函数都正确处理错误并返回有意义的错误信息。

## 关键文件

- [src/utils/bash/ast.ts](src/utils/bash/ast.ts) — Shell AST 解析
- [src/utils/permissions/permissions.ts](src/utils/permissions/permissions.ts) — 权限核心
- [src/utils/model/model.ts](src/utils/model/model.ts) — 模型配置
- [src/utils/git/operations.ts](src/utils/git/operations.ts) — Git 操作

## 关联文档

- [05-工具系统](05-tool-system.md)
- [07-权限系统](07-permission-system.md)
- [14-Hooks清单](14-hooks-catalog.md)