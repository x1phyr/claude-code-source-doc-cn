# 权限检查细节

## 概述

权限检查是 Claude Code 安全模型的核心，决定了每个工具调用是否需要用户确认。系统基于权限模式和规则进行细粒度的访问控制。

## 核心文件

- [src/utils/permissions/permissions.ts](src/utils/permissions/permissions.ts) — 权限检查
- [src/utils/permissions/PermissionRule.ts](src/utils/permissions/PermissionRule.ts) — 权限规则
- [src/utils/permissions/PermissionResult.ts](src/utils/permissions/PermissionResult.ts) — 权限结果

## 检查流程

### 主入口

```typescript
export async function checkPermissions(
  tool: Tool,
  input: unknown,
  context: ToolUseContext,
  canUseTool: CanUseToolFn,
): Promise<PermissionResult> {
  // 1. 检查权限模式
  const modeResult = checkPermissionMode(context)
  if (modeResult) return modeResult

  // 2. 检查始终允许规则
  const allowRule = toolAlwaysAllowedRule(context.toolPermissionContext, tool)
  if (allowRule) {
    return { behavior: 'allow', decisionReason: { type: 'rule', rule: allowRule } }
  }

  // 3. 检查始终拒绝规则
  const denyRule = getDenyRuleForTool(context.toolPermissionContext, tool)
  if (denyRule) {
    return { behavior: 'deny', message: '...', decisionReason: { type: 'rule', rule: denyRule } }
  }

  // 4. 检查始终询问规则
  const askRule = getAskRuleForTool(context.toolPermissionContext, tool)

  // 5. 工具特定检查
  const toolResult = await tool.checkPermissions?.(input, context, canUseTool)

  // 6. 分类器检查（auto 模式）
  if (mode === 'auto') {
    const classifierResult = await runClassifierCheck(tool, input, context)
    if (classifierResult) return classifierResult
  }

  // 7. Hook 检查
  const hookResult = await executePermissionRequestHooks(tool, input)

  // 8. 默认行为
  return { behavior: 'ask', message: createPermissionRequestMessage(tool.name) }
}
```

## 权限模式

### 模式类型

```typescript
type PermissionMode =
  | 'default'        // 敏感操作需要确认
  | 'acceptEdits'    // 编辑操作自动接受
  | 'bypassPermissions' // 所有操作自动允许（危险）
  | 'dontAsk'        // 所有操作自动拒绝
  | 'plan'           // 规划模式权限规则
  | 'auto'           // 分类器自动决定
```

### 模式检查

```typescript
function checkPermissionMode(
  context: ToolUseContext,
): PermissionResult | null {
  const mode = context.toolPermissionContext.mode

  switch (mode) {
    case 'bypassPermissions':
      return { behavior: 'allow' }

    case 'dontAsk':
      return {
        behavior: 'deny',
        message: DONT_ASK_REJECT_MESSAGE,
      }

    case 'acceptEdits':
      // 编辑工具自动允许
      if (isEditTool(tool)) {
        return { behavior: 'allow' }
      }
      break

    case 'plan':
      // 规划模式限制
      return checkPlanModePermissions(tool, input)

    case 'auto':
      // 分类器决定
      return null // 继续检查
  }

  return null
}
```

## 权限规则

### 规则结构

```typescript
type PermissionRule = {
  source: PermissionRuleSource  // 规则来源
  ruleBehavior: PermissionBehavior  // 行为：allow/deny/ask
  ruleValue: PermissionRuleValue  // 规则值
}

type PermissionRuleValue = {
  toolName: string  // 工具名称
  ruleContent?: string  // 规则内容（如 Bash 的命令模式）
}

type PermissionRuleSource =
  | 'userSettings'
  | 'projectSettings'
  | 'localSettings'
  | 'flagSettings'
  | 'policySettings'
  | 'cliArg'
  | 'command'
  | 'session'
```

### 规则匹配

```typescript
// 工具名称匹配
function toolMatchesRule(
  tool: Pick<Tool, 'name' | 'mcpInfo'>,
  rule: PermissionRule,
): boolean {
  // 规则不能有内容才能匹配整个工具
  if (rule.ruleValue.ruleContent !== undefined) {
    return false
  }

  const nameForRuleMatch = getToolNameForPermissionCheck(tool)

  // 直接名称匹配
  if (rule.ruleValue.toolName === nameForRuleMatch) {
    return true
  }

  // MCP 服务器级别权限
  const ruleInfo = mcpInfoFromString(rule.ruleValue.toolName)
  const toolInfo = mcpInfoFromString(nameForRuleMatch)

  return (
    ruleInfo !== null &&
    toolInfo !== null &&
    (ruleInfo.toolName === undefined || ruleInfo.toolName === '*') &&
    ruleInfo.serverName === toolInfo.serverName
  )
}
```

### 规则语法

```
ToolName                    # 匹配整个工具
ToolName(pattern)           # 匹配工具 + 输入模式
mcp__server                 # 匹配 MCP 服务器的所有工具
mcp__server__*              # 同上
```

### 示例规则

```json
{
  "permissions": {
    "allow": [
      "Read",                    // 允许所有读取操作
      "Edit",                    // 允许所有编辑操作
      "Bash(npm run *)"          // 允许 npm run 开头的命令
    ],
    "deny": [
      "Bash(rm -rf /*)",         // 拒绝危险命令
      "Bash(sudo *)"             // 拒绝 sudo 命令
    ],
    "ask": [
      "Bash(git push *)",        // git push 需要确认
      "Write"                    // 写文件需要确认
    ]
  }
}
```

## 权限结果

### 结果类型

```typescript
type PermissionResult<Input = unknown> =
  | { behavior: 'allow'; updatedInput?: Input }
  | { behavior: 'deny'; message: string; updatedInput?: Input }
  | { behavior: 'ask'; message: string }
  | { behavior: 'passthrough'; message: string }
```

### 决策原因

```typescript
type PermissionDecisionReason =
  | { type: 'mode'; mode: PermissionMode }
  | { type: 'rule'; rule: PermissionRule }
  | { type: 'hook'; hookName: string; reason?: string }
  | { type: 'classifier'; classifier: string; reason: string }
  | { type: 'subcommandResults'; reasons: Map<string, PermissionResult> }
  | { type: 'permissionPromptTool'; permissionPromptToolName: string }
  | { type: 'sandboxOverride' }
  | { type: 'workingDir'; reason: string }
  | { type: 'safetyCheck'; reason: string }
  | { type: 'asyncAgent'; reason: string }
  | { type: 'other'; reason: string }
```

## 分类器 (Auto 模式)

### 分类逻辑

```typescript
async function runClassifierCheck(
  tool: Tool,
  input: unknown,
  context: ToolUseContext,
): Promise<PermissionResult | null> {
  if (!feature('TRANSCRIPT_CLASSIFIER')) return null

  const classifierInput = tool.toAutoClassifierInput(input)
  const action = classifyYoloAction(tool.name, classifierInput, context)

  switch (action.type) {
    case 'allow':
      return { behavior: 'allow', decisionReason: { type: 'classifier', ... } }
    case 'deny':
      return { behavior: 'deny', message: action.reason, ... }
    case 'ask':
      return { behavior: 'ask', message: action.reason, ... }
  }
}
```

### 分类器输入

```typescript
// 工具提供分类器输入
interface Tool {
  toAutoClassifierInput(input: unknown): unknown
}

// Bash 工具示例
toAutoClassifierInput(input) {
  return {
    command: input.command,
    timeout: input.timeout,
    // 不包含敏感数据如文件内容
  }
}
```

## Bash 特殊检查

### 子命令解析

```typescript
// 复合命令分解
const subcommands = parseSubcommands(input.command)

// 逐个检查
const results = new Map<string, PermissionResult>()
for (const cmd of subcommands) {
  const result = await checkSingleBashCommand(cmd, context)
  results.set(cmd, result)
}

// 汇总结果
if (allAllowed(results)) {
  return { behavior: 'allow' }
} else if (anyDenied(results)) {
  return { behavior: 'deny', message: deniedMessages(results) }
} else {
  return {
    behavior: 'ask',
    message: createSubcommandRequestMessage(results),
    decisionReason: { type: 'subcommandResults', reasons: results }
  }
}
```

### 沙箱检查

```typescript
// 检查是否应该使用沙箱
if (shouldUseSandbox(tool, input, context)) {
  // 沙箱内的命令自动允许
  if (SandboxManager.isSandboxingEnabled() && SandboxManager.isAutoAllowBashIfSandboxedEnabled()) {
    return {
      behavior: 'allow',
      decisionReason: { type: 'sandboxOverride' }
    }
  }
}
```

## 拒绝追踪

### 追踪状态

```typescript
interface DenialTrackingState {
  denials: Map<string, { count: number; lastDenialTime: number }>
}

const DENIAL_LIMITS = {
  maxDenials: 3,
  windowMs: 60000,  // 1 分钟
}

function recordDenial(state: DenialTrackingState, key: string): void {
  const current = state.denials.get(key) || { count: 0, lastDenialTime: 0 }
  state.denials.set(key, {
    count: current.count + 1,
    lastDenialTime: Date.now(),
  })
}

function shouldFallbackToPrompting(state: DenialTrackingState, key: string): boolean {
  const denial = state.denials.get(key)
  if (!denial) return false
  return denial.count >= DENIAL_LIMITS.maxDenials
}
```

## 消息生成

### 请求消息

```typescript
export function createPermissionRequestMessage(
  toolName: string,
  decisionReason?: PermissionDecisionReason,
): string {
  if (decisionReason) {
    switch (decisionReason.type) {
      case 'classifier':
        return `Classifier '${decisionReason.classifier}' requires approval: ${decisionReason.reason}`

      case 'hook':
        return decisionReason.reason
          ? `Hook '${decisionReason.hookName}' blocked: ${decisionReason.reason}`
          : `Hook '${decisionReason.hookName}' requires approval`

      case 'rule': {
        const ruleString = permissionRuleValueToString(decisionReason.rule.ruleValue)
        const sourceString = permissionRuleSourceDisplayString(decisionReason.rule.source)
        return `Permission rule '${ruleString}' from ${sourceString} requires approval`
      }

      case 'mode': {
        const modeTitle = permissionModeTitle(decisionReason.mode)
        return `Current permission mode (${modeTitle}) requires approval`
      }
      // ...
    }
  }

  return `Claude requested permissions to use ${toolName}, but you haven't granted it yet.`
}
```

## 关键文件

- [src/utils/permissions/permissions.ts](src/utils/permissions/permissions.ts) — 权限检查
- [src/utils/permissions/PermissionRule.ts](src/utils/permissions/PermissionRule.ts) — 规则定义
- [src/utils/permissions/PermissionResult.ts](src/utils/permissions/PermissionResult.ts) — 结果类型
- [src/utils/permissions/yoloClassifier.ts](src/utils/permissions/yoloClassifier.ts) — 分类器
- [src/utils/permissions/denialTracking.ts](src/utils/permissions/denialTracking.ts) — 拒绝追踪

## 关联文档

- [05-权限安全/权限系统详解-Permission-System.md](../05-权限安全/权限系统详解-Permission-System.md)
- [03-工具系统/沙箱实现-Sandbox-Implementation.md](沙箱实现-Sandbox-Implementation.md)