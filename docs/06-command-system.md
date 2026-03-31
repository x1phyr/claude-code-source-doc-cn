# 命令系统详解

## 概述

命令系统管理 Claude Code 的所有斜杠命令（如 `/help`、`/config`、`/clear`）。命令分为三种类型：`prompt`、`local`、`local-jsx`。

## Command 类型

### 定义位置

`src/types/command.ts`

### 核心结构

```typescript
type Command =
  | PromptCommand
  | LocalCommand
  | LocalJSXCommand

type CommandBase = {
  name: string                    // 命令名称
  aliases?: string[]              // 别名
  description: string             // 描述
  availability?: ('claude-ai' | 'console')[]  // 可用性要求
  isEnabled?: () => boolean       // 是否启用
}

// prompt 类型：展开为发送给模型的文本
type PromptCommand = CommandBase & {
  type: 'prompt'
  contentLength: number           // 内容长度
  progressMessage?: string        // 进度消息
  source: 'builtin' | 'plugin' | 'skills' | 'bundled' | 'mcp' | 'commands_DEPRECATED'
  loadedFrom?: string             // 加载来源
  hasUserSpecifiedDescription?: boolean  // 用户指定描述
  whenToUse?: string              // 使用时机
  kind?: 'workflow'               // 特殊种类
  pluginInfo?: {                  // 插件信息
    pluginManifest: { name: string; ... }
  }
  getPromptForCommand(args, context): Promise<string>
}

// local 类型：本地执行，返回文本输出
type LocalCommand = CommandBase & {
  type: 'local'
  handler(args, context): Promise<LocalCommandResult>
}

// local-jsx 类型：渲染 Ink UI 组件
type LocalJSXCommand = CommandBase & {
  type: 'local-jsx'
  handler(args, context): Promise<React.ReactNode>
}
```

## 命令类型说明

### prompt 类型

展开为文本，发送给模型处理：

```typescript
const commit: Command = {
  type: 'prompt',
  name: 'commit',
  description: 'Create a git commit',
  source: 'builtin',
  async getPromptForCommand(args, context) {
    return 'Create a git commit with the following changes...'
  }
}
```

**适用场景**：
- Skills（技能）
- Workflows（工作流）
- 需要模型处理的命令

### local 类型

本地执行，返回文本输出：

```typescript
const clear: Command = {
  type: 'local',
  name: 'clear',
  description: 'Clear the screen',
  async handler(args, context) {
    return { output: 'Screen cleared' }
  }
}
```

**适用场景**：
- 简单的本地操作
- 不需要模型处理的命令

### local-jsx 类型

渲染 Ink UI 组件：

```typescript
const config: Command = {
  type: 'local-jsx',
  name: 'config',
  description: 'Manage settings',
  async handler(args, context) {
    return <ConfigUI />
  }
}
```

**适用场景**：
- 需要交互式 UI 的命令
- 复杂的配置界面

## 命令注册

### 定义位置

`src/commands.ts`

### 命令列表

```typescript
const COMMANDS = memoize((): Command[] => [
  addDir,
  advisor,
  agents,
  branch,
  btw,
  chrome,
  clear,
  color,
  compact,
  config,
  // ... 更多命令
])
```

### 命令加载

命令从多个来源加载：

```typescript
async function getSkills(cwd: string) {
  const [skillDirCommands, pluginSkills, bundledSkills, builtinPluginSkills] =
    await Promise.all([
      getSkillDirCommands(cwd),      // 从 skills/ 目录
      getPluginSkills(),              // 从插件
      getBundledSkills(),             // 内置技能
      getBuiltinPluginSkillCommands(), // 内置插件技能
    ])
  return { skillDirCommands, pluginSkills, bundledSkills, builtinPluginSkills }
}

export async function getCommands(cwd: string): Promise<Command[]> {
  const allCommands = await loadAllCommands(cwd)
  return allCommands.filter(cmd =>
    meetsAvailabilityRequirement(cmd) &&
    isCommandEnabled(cmd)
  )
}
```

### 加载来源

| source | 说明 |
|--------|------|
| `builtin` | 内置命令 |
| `bundled` | 打包的技能 |
| `skills` | skills/ 目录中的技能 |
| `plugin` | 插件提供的命令 |
| `mcp` | MCP 服务器提供的命令 |
| `commands_DEPRECATED` | 旧版命令目录 |

## 命令过滤

### 可用性过滤

```typescript
export function meetsAvailabilityRequirement(cmd: Command): boolean {
  if (!cmd.availability) return true
  for (const a of cmd.availability) {
    switch (a) {
      case 'claude-ai':
        if (isClaudeAISubscriber()) return true
        break
      case 'console':
        if (!isClaudeAISubscriber() && !isUsing3PServices() && isFirstPartyAnthropicBaseUrl())
          return true
        break
    }
  }
  return false
}
```

### 启用状态过滤

```typescript
export function isCommandEnabled(cmd: Command): boolean {
  return cmd.isEnabled?.() ?? true
}
```

### 远程模式过滤

```typescript
export const REMOTE_SAFE_COMMANDS: Set<Command> = new Set([
  session, exit, clear, help, theme, color, vim, cost, usage, copy, btw, feedback, plan, keybindings, statusline, stickers, mobile
])

export function filterCommandsForRemoteMode(commands: Command[]): Command[] {
  return commands.filter(cmd => REMOTE_SAFE_COMMANDS.has(cmd))
}
```

## 命令分类

### 会话管理

| 命令 | 类型 | 说明 |
|------|------|------|
| `/resume` | prompt | 恢复会话 |
| `/session` | local-jsx | 会话管理 |
| `/clear` | local | 清屏 |
| `/compact` | local | 上下文压缩 |
| `/rewind` | local-jsx | 回滚操作 |

### 配置

| 命令 | 类型 | 说明 |
|------|------|------|
| `/config` | local-jsx | 配置管理 |
| `/permissions` | local-jsx | 权限管理 |
| `/hooks` | local-jsx | 钩子管理 |
| `/mcp` | local-jsx | MCP 管理 |
| `/model` | local-jsx | 模型选择 |

### Git 操作

| 命令 | 类型 | 说明 |
|------|------|------|
| `/commit` | prompt | 创建提交 |
| `/branch` | local-jsx | 分支操作 |
| `/diff` | local | 差异查看 |
| `/pr_comments` | prompt | PR 评论 |

### 工具

| 命令 | 类型 | 说明 |
|------|------|------|
| `/doctor` | local | 诊断工具 |
| `/help` | local-jsx | 帮助信息 |
| `/status` | local | 状态显示 |
| `/cost` | local | 成本统计 |
| `/usage` | local | 使用统计 |

### 开发

| 命令 | 类型 | 说明 |
|------|------|------|
| `/init` | prompt | 初始化项目 |
| `/upgrade` | local-jsx | 升级 CLI |

### 其他

| 命令 | 类型 | 说明 |
|------|------|------|
| `/memory` | local-jsx | 记忆系统 |
| `/skills` | local-jsx | 技能管理 |
| `/theme` | local-jsx | 主题管理 |
| `/vim` | local | Vim 模式 |
| `/fast` | local | 快速模式 |

## 命令执行流程

```
用户输入 "/command args"
    ↓
processUserInput() 解析
    ↓
findCommand() 查找命令
    ↓
根据类型执行：
├── prompt → getPromptForCommand() → 发送给模型
├── local → handler() → 返回文本
└── local-jsx → handler() → 渲染 UI
```

## 技能目录结构

```
skills/
├── my-skill/
│   ├── SKILL.md          # 技能定义
│   └── examples/
│       └── example.md
```

### SKILL.md 格式

```markdown
---
name: my-skill
description: My skill description
whenToUse: When you need to do X
---

# My Skill

Detailed instructions...
```

## 内置技能

定义在 `src/skills/bundled/`：

| 技能 | 说明 |
|------|------|
| `claude-api` | Claude API 文档 |
| `verify` | 验证技能 |
| `keybindings` | 快捷键帮助 |
| `loop` | 循环执行 |
| `simplify` | 代码简化 |
| `remember` | 记忆管理 |
| `stuck` | 卡住时帮助 |
| `scheduleRemoteAgents` | 调度远程代理 |

## 关键文件

- `src/commands.ts` - 命令注册中心
- `src/types/command.ts` - Command 类型定义
- `src/commands/` - 各命令实现
- `src/skills/bundledSkills.ts` - 内置技能注册
- `src/skills/loadSkillsDir.ts` - 技能目录加载

## 关联文档

- [05-工具系统](05-tool-system.md)
- [11-命令清单](11-commands-catalog.md)
- [16-Skills系统](16-skills-system.md)