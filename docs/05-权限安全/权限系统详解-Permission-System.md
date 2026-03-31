# 权限系统详解

## 概述

权限系统控制 Claude Code 执行敏感操作的许可。它支持多种权限模式、规则配置和自动分类器。

## 权限模式

### 定义位置

`src/types/permissions.ts`

### 模式类型

```typescript
type PermissionMode =
  | 'default'          // 默认模式，需要用户确认
  | 'acceptEdits'      // 自动接受编辑操作
  | 'bypassPermissions' // 绕过所有权限检查
  | 'dontAsk'          // 不询问（自动拒绝）
  | 'plan'             // 规划模式
  | 'auto'             // 自动模式（使用分类器）
```

### 模式说明

| 模式 | 行为 |
|------|------|
| `default` | 敏感操作需要用户确认 |
| `acceptEdits` | 文件编辑自动接受，其他操作正常检查 |
| `bypassPermissions` | 所有操作自动允许（危险） |
| `dontAsk` | 所有操作自动拒绝 |
| `plan` | 规划模式下的权限规则 |
| `auto` | 使用分类器自动决定 |

## 权限规则

### 规则结构

```typescript
type PermissionRule = {
  source: PermissionRuleSource    // 规则来源
  ruleBehavior: PermissionBehavior  // 行为
  ruleValue: PermissionRuleValue    // 规则值
}

type PermissionRuleValue = {
  toolName: string           // 工具名称
  ruleContent?: string       // 规则内容（如 "git *"）
}

type PermissionBehavior = 'allow' | 'deny' | 'ask'
```

### 规则来源

```typescript
type PermissionRuleSource =
  | 'userSettings'      // 用户设置
  | 'projectSettings'   // 项目设置
  | 'localSettings'     // 本地设置
  | 'flagSettings'      // 标志设置
  | 'policySettings'    // 策略设置
  | 'cliArg'           // 命令行参数
  | 'command'          // 命令触发
  | 'session'          // 会话级别
```

### 规则配置示例

```json
{
  "allowedTools": ["Read", "Edit", "Write"],
  "deniedTools": ["Bash(rm -rf *)"],
  "askTools": ["Bash(git push *)"]
}
```

## ToolPermissionContext

### 结构

```typescript
type ToolPermissionContext = {
  mode: PermissionMode                              // 权限模式
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>  // 额外工作目录
  alwaysAllowRules: ToolPermissionRulesBySource    // 总是允许规则
  alwaysDenyRules: ToolPermissionRulesBySource     // 总是拒绝规则
  alwaysAskRules: ToolPermissionRulesBySource      // 总是询问规则
  isBypassPermissionsModeAvailable: boolean       // 绕过模式是否可用
  strippedDangerousRules?: ToolPermissionRulesBySource  // 剥离的危险规则
  shouldAvoidPermissionPrompts?: boolean           // 避免权限提示
  awaitAutomatedChecksBeforeDialog?: boolean       // 等待自动检查
  prePlanMode?: PermissionMode                     // 规划前模式
}
```

## 权限决策

### 决策结果

```typescript
type PermissionDecision<Input> =
  | { behavior: 'allow'; updatedInput?: Input; ... }   // 允许
  | { behavior: 'ask'; message: string; ... }          // 询问用户
  | { behavior: 'deny'; message: string; ... }         // 拒绝

type PermissionResult<Input> =
  | PermissionDecision<Input>
  | { behavior: 'passthrough'; message: string; ... }  // 传递
```

### 决策原因

```typescript
type PermissionDecisionReason =
  | { type: 'rule'; rule: PermissionRule }           // 规则触发
  | { type: 'mode'; mode: PermissionMode }           // 模式决定
  | { type: 'subcommandResults'; reasons: Map<...> } // 子命令结果
  | { type: 'hook'; hookName: string; ... }          // Hook 决定
  | { type: 'classifier'; classifier: string; ... }  // 分类器决定
  | { type: 'workingDir'; reason: string }           // 工作目录决定
  | { type: 'safetyCheck'; reason: string; ... }     // 安全检查
  | { type: 'other'; reason: string }                // 其他原因
```

## 权限检查流程

```
工具调用请求
    ↓
┌──────────────────┐
│  规则匹配检查     │  alwaysAllow/alwaysDeny/alwaysAsk
└──────────────────┘
    │
    ├─ 匹配 deny → 拒绝
    ├─ 匹配 allow → 允许
    ├─ 匹配 ask → 询问用户
    │
    ↓ 无匹配规则
┌──────────────────┐
│  模式检查        │  bypassPermissions/acceptEdits 等
└──────────────────┘
    │
    ├─ bypass → 允许
    ├─ dontAsk → 拒绝
    │
    ↓ 需要检查
┌──────────────────┐
│  工具权限检查     │  tool.checkPermissions()
└──────────────────┘
    │
    ↓
┌──────────────────┐
│  自动模式分类器   │  YoloClassifier
└──────────────────┘
    │
    ↓ 无法自动决定
┌──────────────────┐
│  询问用户        │  PermissionDialog
└──────────────────┘
```

## 自动分类器

### YoloClassifier

用于 `auto` 模式，自动决定是否允许操作：

```typescript
type YoloClassifierResult = {
  thinking?: string          // 思考过程
  shouldBlock: boolean       // 是否阻止
  reason: string             // 原因
  unavailable?: boolean      // 是否不可用
  transcriptTooLong?: boolean // 是否超出上下文
  model: string              // 使用的模型
  usage?: ClassifierUsage    // Token 使用量
  durationMs?: number        // 耗时
  stage?: 'fast' | 'thinking' // 分类阶段
}
```

### 分类流程

```
1. 快速阶段 (fast)
   - 使用轻量模型快速判断
   - 高置信度直接返回

2. 思考阶段 (thinking)
   - 快速阶段不确定时触发
   - 使用更强模型深入分析
   - 返回最终决策
```

## 沙箱模式

### 概述

沙箱模式在隔离环境中执行命令，提供额外安全层。

### 相关配置

- 环境变量控制沙箱启用
- 危险命令自动被阻止
- 特定目录被保护

## 权限更新

### 更新操作

```typescript
type PermissionUpdate =
  | { type: 'addRules'; destination: ...; rules: ...; behavior: ... }
  | { type: 'replaceRules'; destination: ...; rules: ...; behavior: ... }
  | { type: 'removeRules'; destination: ...; rules: ...; behavior: ... }
  | { type: 'setMode'; destination: ...; mode: ... }
  | { type: 'addDirectories'; destination: ...; directories: ... }
  | { type: 'removeDirectories'; destination: ...; directories: ... }
```

### 更新目标

```typescript
type PermissionUpdateDestination =
  | 'userSettings'      // 用户级设置
  | 'projectSettings'   // 项目级设置
  | 'localSettings'     // 本地设置
  | 'session'           // 会话级
  | 'cliArg'            // 命令行参数
```

## Hook 集成

权限系统与 Hook 系统集成：

- `PreToolUse` hook 在工具执行前运行
- Hook 可以修改权限决策
- Hook 可以提供额外上下文

## 关键文件

- `src/types/permissions.ts` - 权限类型定义
- `src/utils/permissions/` - 权限实现
- `src/hooks/useCanUseTool.tsx` - 权限检查 hook
- `src/hooks/toolPermission/` - 工具权限 handlers

## 关联文档

- [05-工具系统](05-tool-system.md)
- [18-Hook系统](18-hooks-system.md)
- [23-配置参考](23-configuration-reference.md)