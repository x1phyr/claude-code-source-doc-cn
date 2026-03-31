# Hook 系统详解（用户自定义）

## 概述

Hook 系统允许用户在特定事件点注入自定义逻辑。这与 React Hooks 不同，是 CLI 层面的事件钩子机制。

## Hook 类型

### 定义位置

[src/utils/hooks/](src/utils/hooks/)

```typescript
type HookType =
  | 'PreToolUse'       // 工具执行前
  | 'PostToolUse'      // 工具执行后
  | 'Stop'             // 会话停止时
  | 'Notification'     // 通知触发时
  | 'PrePrompt'        // 提示发送前
```

## Hook 配置

### 配置格式

在 `settings.json` 中配置：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": ["echo 'Before Bash'"]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": ["echo 'After Bash'"]
      }
    ],
    "Stop": [
      {
        "matcher": "",
        "hooks": ["echo 'Session stopped'"]
      }
    ]
  }
}
```

### Matcher 规则

```typescript
type HookMatcher =
  | string              // 工具名称匹配
  | { tool: string }    // 指定工具
  | { input: RegExp }   // 输入匹配
```

## Hook 执行

### 执行流程

```
事件触发
    ↓
匹配 Hook 规则
    ↓
执行 Hook 命令
    ↓
处理 Hook 结果
    ↓
继续/中断 原操作
```

### PreToolUse Hook

```typescript
export async function runPreToolUseHook(
  tool: Tool,
  input: unknown,
  context: HookContext
): Promise<HookResult> {
  const hooks = getHooksForTool('PreToolUse', tool.name)

  for (const hook of hooks) {
    const result = await executeHook(hook, {
      tool: tool.name,
      input: JSON.stringify(input)
    })

    if (result.block) {
      return { block: true, reason: result.reason }
    }

    if (result.modifiedInput) {
      input = JSON.parse(result.modifiedInput)
    }
  }

  return { block: false, modifiedInput: input }
}
```

### PostToolUse Hook

```typescript
export async function runPostToolUseHook(
  tool: Tool,
  input: unknown,
  output: unknown,
  context: HookContext
): Promise<void> {
  const hooks = getHooksForTool('PostToolUse', tool.name)

  for (const hook of hooks) {
    await executeHook(hook, {
      tool: tool.name,
      input: JSON.stringify(input),
      output: JSON.stringify(output)
    })
  }
}
```

### Stop Hook

```typescript
export async function runStopHook(
  reason: StopReason,
  context: HookContext
): Promise<void> {
  const hooks = getHooks('Stop')

  for (const hook of hooks) {
    await executeHook(hook, {
      reason
    })
  }
}
```

## Hook 上下文

```typescript
type HookContext = {
  sessionId: string
  cwd: string
  timestamp: number
  toolName?: string
  input?: unknown
  output?: unknown
}
```

## Hook 结果

```typescript
type HookResult = {
  block?: boolean           // 是否阻止原操作
  reason?: string           // 阻止原因
  modifiedInput?: string    // 修改后的输入
  output?: string           // Hook 输出
}
```

## 命令行 Hook

### /hooks 命令

管理 Hook 配置：

```typescript
const hooksCommand = {
  type: 'local-jsx',
  name: 'hooks',
  description: 'Manage hooks',
  handler: async () => {
    return <HooksManager />
  }
}
```

### HooksManager 组件

```tsx
function HooksManager() {
  const hooks = useSettings(s => s.hooks)

  return (
    <Box flexDirection="column">
      <Text bold>Hooks Configuration</Text>
      {Object.entries(hooks).map(([type, hookList]) => (
        <Box key={type}>
          <Text>{type}: {hookList.length} hooks</Text>
        </Box>
      ))}
    </Box>
  )
}
```

## 环境变量

Hook 命令执行时的环境变量：

| 变量 | 说明 |
|------|------|
| `CLAUDE_TOOL_NAME` | 工具名称 |
| `CLAUDE_TOOL_INPUT` | 工具输入（JSON） |
| `CLAUDE_TOOL_OUTPUT` | 工具输出（JSON） |
| `CLAUDE_SESSION_ID` | 会话 ID |
| `CLAUDE_CWD` | 工作目录 |

## 示例配置

### 自动格式化

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

### 安全检查

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": ["security-check $COMMAND"]
      }
    ]
  }
}
```

### 日志记录

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "",
        "hooks": ["log-tool-use $TOOL_NAME"]
      }
    ]
  }
}
```

## 关键文件

- [src/utils/hooks/postSamplingHooks.ts](src/utils/hooks/postSamplingHooks.ts) — 采样后 Hooks
- [src/utils/hooks/sessionHooks.ts](src/utils/hooks/sessionHooks.ts) — 会话 Hooks
- [src/commands/hooks/index.js](src/commands/hooks/index.js) — /hooks 命令

## 关联文档

- [07-权限系统](07-permission-system.md)
- [23-配置参考](23-configuration-reference.md)