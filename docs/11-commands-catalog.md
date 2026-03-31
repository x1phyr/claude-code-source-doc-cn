# 命令清单

## 概述

本文档详细列出 Claude Code 所有内置斜杠命令的定义、类型和来源。

## 命令注册

### 定义位置

[src/commands.ts](src/commands.ts)

### 命令获取流程

```typescript
const COMMANDS = memoize((): Command[] => [
  addDir,
  advisor,
  agents,
  branch,
  // ... 更多命令
])

export async function getCommands(cwd: string): Promise<Command[]> {
  const allCommands = await loadAllCommands(cwd)

  // 过滤可用性和启用状态
  return allCommands.filter(
    _ => meetsAvailabilityRequirement(_) && isCommandEnabled(_)
  )
}

async function loadAllCommands(cwd: string): Promise<Command[]> {
  const [
    { skillDirCommands, pluginSkills, bundledSkills, builtinPluginSkills },
    pluginCommands,
    workflowCommands,
  ] = await Promise.all([
    getSkills(cwd),
    getPluginCommands(),
    getWorkflowCommands?.(cwd),
  ])

  return [
    ...bundledSkills,
    ...builtinPluginSkills,
    ...skillDirCommands,
    ...workflowCommands,
    ...pluginCommands,
    ...pluginSkills,
    ...COMMANDS(),
  ]
}
```

## 命令类型

### prompt 类型

展开为文本，发送给模型处理。

```typescript
type PromptCommand = {
  type: 'prompt'
  name: string
  description: string
  contentLength: number
  progressMessage?: string
  source: 'builtin' | 'plugin' | 'skills' | 'bundled' | 'mcp'
  getPromptForCommand(args, context): Promise<string>
}
```

### local 类型

本地执行，返回文本输出。

```typescript
type LocalCommand = {
  type: 'local'
  name: string
  description: string
  handler(args, context): Promise<LocalCommandResult>
}
```

### local-jsx 类型

渲染 Ink UI 组件。

```typescript
type LocalJSXCommand = {
  type: 'local-jsx'
  name: string
  description: string
  handler(args, context): Promise<React.ReactNode>
}
```

## 会话管理命令

### /resume

| 属性 | 值 |
|------|------|
| name | `resume` |
| type | `prompt` |
| source | `builtin` |
| 文件 | [src/commands/resume/index.js](src/commands/resume/index.js) |

**说明**: 恢复之前的会话。

### /session

| 属性 | 值 |
|------|------|
| name | `session` |
| type | `local-jsx` |
| source | `builtin` |
| 文件 | [src/commands/session/index.js](src/commands/session/index.js) |

**说明**: 会话管理界面。

### /clear

| 属性 | 值 |
|------|------|
| name | `clear` |
| type | `local` |
| source | `builtin` |
| 文件 | [src/commands/clear/index.js](src/commands/clear/index.js) |

**说明**: 清屏。

### /compact

| 属性 | 值 |
|------|------|
| name | `compact` |
| type | `local` |
| source | `builtin` |
| 文件 | [src/commands/compact/index.js](src/commands/compact/index.js) |

**说明**: 压缩上下文。

### /rewind

| 属性 | 值 |
|------|------|
| name | `rewind` |
| type | `local-jsx` |
| source | `builtin` |
| 文件 | [src/commands/rewind/index.js](src/commands/rewind/index.js) |

**说明**: 回滚操作。

### /export

| 属性 | 值 |
|------|------|
| name | `export` |
| type | `local-jsx` |
| source | `builtin` |
| 文件 | [src/commands/export/index.js](src/commands/export/index.js) |

**说明**: 导出会话。

## 配置命令

### /config

| 属性 | 值 |
|------|------|
| name | `config` |
| type | `local-jsx` |
| source | `builtin` |
| 文件 | [src/commands/config/index.js](src/commands/config/index.js) |

**说明**: 配置管理界面。

### /permissions

| 属性 | 值 |
|------|------|
| name | `permissions` |
| type | `local-jsx` |
| source | `builtin` |
| 文件 | [src/commands/permissions/index.js](src/commands/permissions/index.js) |

**说明**: 权限管理界面。

### /hooks

| 属性 | 值 |
|------|------|
| name | `hooks` |
| type | `local-jsx` |
| source | `builtin` |
| 文件 | [src/commands/hooks/index.js](src/commands/hooks/index.js) |

**说明**: Hook 管理界面。

### /mcp

| 属性 | 值 |
|------|------|
| name | `mcp` |
| type | `local-jsx` |
| source | `builtin` |
| 文件 | [src/commands/mcp/index.js](src/commands/mcp/index.js) |

**说明**: MCP 服务器管理界面。

### /model

| 属性 | 值 |
|------|------|
| name | `model` |
| type | `local-jsx` |
| source | `builtin` |
| 文件 | [src/commands/model/index.js](src/commands/model/index.js) |

**说明**: 模型选择界面。

### /theme

| 属性 | 值 |
|------|------|
| name | `theme` |
| type | `local-jsx` |
| source | `builtin` |
| 文件 | [src/commands/theme/index.js](src/commands/theme/index.js) |

**说明**: 主题管理。

### /output-style

| 属性 | 值 |
|------|------|
| name | `output-style` |
| type | `local-jsx` |
| source | `builtin` |
| 文件 | [src/commands/output-style/index.js](src/commands/output-style/index.js) |

**说明**: 输出风格配置。

### /effort

| 属性 | 值 |
|------|------|
| name | `effort` |
| type | `local-jsx` |
| source | `builtin` |
| 文件 | [src/commands/effort/index.js](src/commands/effort/index.js) |

**说明**: Effort 值设置。

### /rate-limit-options

| 属性 | 值 |
|------|------|
| name | `rate-limit-options` |
| type | `local-jsx` |
| source | `builtin` |
| 文件 | [src/commands/rate-limit-options/index.js](src/commands/rate-limit-options/index.js) |

**说明**: 速率限制配置。

### /privacy-settings

| 属性 | 值 |
|------|------|
| name | `privacy-settings` |
| type | `local-jsx` |
| source | `builtin` |
| 文件 | [src/commands/privacy-settings/index.js](src/commands/privacy-settings/index.js) |

**说明**: 隐私设置。

## Git 操作命令

### /commit

| 属性 | 值 |
|------|------|
| name | `commit` |
| type | `prompt` |
| source | `builtin` |
| 文件 | [src/commands/commit.js](src/commands/commit.js) |
| 条件 | `USER_TYPE === 'ant'` |

**说明**: 创建 Git 提交。

### /branch

| 属性 | 值 |
|------|------|
| name | `branch` |
| type | `local-jsx` |
| source | `builtin` |
| 文件 | [src/commands/branch/index.js](src/commands/branch/index.js) |

**说明**: 分支操作界面。

### /diff

| 属性 | 值 |
|------|------|
| name | `diff` |
| type | `local` |
| source | `builtin` |
| 文件 | [src/commands/diff/index.js](src/commands/diff/index.js) |

**说明**: 查看差异。

### /pr_comments

| 属性 | 值 |
|------|------|
| name | `pr_comments` |
| type | `prompt` |
| source | `builtin` |
| 文件 | [src/commands/pr_comments/index.js](src/commands/pr_comments/index.js) |

**说明**: 获取 PR 评论。

### /review

| 属性 | 值 |
|------|------|
| name | `review` |
| type | `prompt` |
| source | `builtin` |
| 文件 | [src/commands/review.js](src/commands/review.js) |

**说明**: 代码审查。

### /autofix-pr

| 属性 | 值 |
|------|------|
| name | `autofix-pr` |
| type | `prompt` |
| source | `builtin` |
| 文件 | [src/commands/autofix-pr/index.js](src/commands/autofix-pr/index.js) |
| 条件 | `USER_TYPE === 'ant'` |

**说明**: 自动修复 PR 问题。

## 工具命令

### /doctor

| 属性 | 值 |
|------|------|
| name | `doctor` |
| type | `local` |
| source | `builtin` |
| 文件 | [src/commands/doctor/index.js](src/commands/doctor/index.js) |

**说明**: 诊断工具。

### /help

| 属性 | 值 |
|------|------|
| name | `help` |
| type | `local-jsx` |
| source | `builtin` |
| 文件 | [src/commands/help/index.js](src/commands/help/index.js) |

**说明**: 帮助界面。

### /status

| 属性 | 值 |
|------|------|
| name | `status` |
| type | `local` |
| source | `builtin` |
| 文件 | [src/commands/status/index.js](src/commands/status/index.js) |

**说明**: 状态显示。

### /cost

| 属性 | 值 |
|------|------|
| name | `cost` |
| type | `local` |
| source | `builtin` |
| 文件 | [src/commands/cost/index.js](src/commands/cost/index.js) |

**说明**: 成本统计。

### /usage

| 属性 | 值 |
|------|------|
| name | `usage` |
| type | `local` |
| source | `builtin` |
| 文件 | [src/commands/usage/index.js](src/commands/usage/index.js) |

**说明**: 使用统计。

### /stats

| 属性 | 值 |
|------|------|
| name | `stats` |
| type | `local` |
| source | `builtin` |
| 文件 | [src/commands/stats/index.js](src/commands/stats/index.js) |

**说明**: 统计信息。

### /insights

| 属性 | 值 |
|------|------|
| name | `insights` |
| type | `prompt` |
| source | `builtin` |
| 文件 | [src/commands/insights.js](src/commands/insights.js) |

**说明**: 会话分析报告。

## 开发命令

### /init

| 属性 | 值 |
|------|------|
| name | `init` |
| type | `prompt` |
| source | `builtin` |
| 文件 | [src/commands/init.js](src/commands/init.js) |

**说明**: 初始化项目 CLAUDE.md。

### /upgrade

| 属性 | 值 |
|------|------|
| name | `upgrade` |
| type | `local-jsx` |
| source | `builtin` |
| 文件 | [src/commands/upgrade/index.js](src/commands/upgrade/index.js) |

**说明**: 升级 CLI。

### /init-verifiers

| 属性 | 值 |
|------|------|
| name | `init-verifiers` |
| type | `prompt` |
| source | `builtin` |
| 文件 | [src/commands/init-verifiers.js](src/commands/init-verifiers.js) |
| 条件 | `USER_TYPE === 'ant'` |

**说明**: 初始化验证器。

## 认证命令

### /login

| 属性 | 值 |
|------|------|
| name | `login` |
| type | `local-jsx` |
| source | `builtin` |
| 文件 | [src/commands/login/index.js](src/commands/login/index.js) |
| 条件 | `!isUsing3PServices()` |

**说明**: 登录。

### /logout

| 属性 | 值 |
|------|------|
| name | `logout` |
| type | `local` |
| source | `builtin` |
| 文件 | [src/commands/logout/index.js](src/commands/logout/index.js) |
| 条件 | `!isUsing3PServices()` |

**说明**: 登出。

## 技能/插件命令

### /skills

| 属性 | 值 |
|------|------|
| name | `skills` |
| type | `local-jsx` |
| source | `builtin` |
| 文件 | [src/commands/skills/index.js](src/commands/skills/index.js) |

**说明**: 技能管理界面。

### /plugin

| 属性 | 值 |
|------|------|
| name | `plugin` |
| type | `local-jsx` |
| source | `builtin` |
| 文件 | [src/commands/plugin/index.js](src/commands/plugin/index.js) |

**说明**: 插件管理界面。

### /reload-plugins

| 属性 | 值 |
|------|------|
| name | `reload-plugins` |
| type | `local` |
| source | `builtin` |
| 文件 | [src/commands/reload-plugins/index.js](src/commands/reload-plugins/index.js) |

**说明**: 重新加载插件。

### /agents

| 属性 | 值 |
|------|------|
| name | `agents` |
| type | `local-jsx` |
| source | `builtin` |
| 文件 | [src/commands/agents/index.js](src/commands/agents/index.js) |

**说明**: 代理管理界面。

## 其他命令

### /memory

| 属性 | 值 |
|------|------|
| name | `memory` |
| type | `local-jsx` |
| source | `builtin` |
| 文件 | [src/commands/memory/index.js](src/commands/memory/index.js) |

**说明**: 记忆系统界面。

### /vim

| 属性 | 值 |
|------|------|
| name | `vim` |
| type | `local` |
| source | `builtin` |
| 文件 | [src/commands/vim/index.js](src/commands/vim/index.js) |

**说明**: Vim 模式切换。

### /fast

| 属性 | 值 |
|------|------|
| name | `fast` |
| type | `local` |
| source | `builtin` |
| 文件 | [src/commands/fast/index.js](src/commands/fast/index.js) |

**说明**: 快速模式切换。

### /plan

| 属性 | 值 |
|------|------|
| name | `plan` |
| type | `local-jsx` |
| source | `builtin` |
| 文件 | [src/commands/plan/index.js](src/commands/plan/index.js) |

**说明**: 规划模式。

### /files

| 属性 | 值 |
|------|------|
| name | `files` |
| type | `local` |
| source | `builtin` |
| 文件 | [src/commands/files/index.js](src/commands/files/index.js) |

**说明**: 文件列表。

### /context

| 属性 | 值 |
|------|------|
| name | `context` |
| type | `local-jsx` |
| source | `builtin` |
| 文件 | [src/commands/context/index.js](src/commands/context/index.js) |

**说明**: 上下文管理。

### /tasks

| 属性 | 值 |
|------|------|
| name | `tasks` |
| type | `local-jsx` |
| source | `builtin` |
| 文件 | [src/commands/tasks/index.js](src/commands/tasks/index.js) |

**说明**: 任务管理。

### /add-dir

| 属性 | 值 |
|------|------|
| name | `add-dir` |
| type | `local-jsx` |
| source | `builtin` |
| 文件 | [src/commands/add-dir/index.js](src/commands/add-dir/index.js) |

**说明**: 添加工作目录。

### /tag

| 属性 | 值 |
|------|------|
| name | `tag` |
| type | `local-jsx` |
| source | `builtin` |
| 文件 | [src/commands/tag/index.js](src/commands/tag/index.js) |

**说明**: 标签管理。

### /keybindings

| 属性 | 值 |
|------|------|
| name | `keybindings` |
| type | `local-jsx` |
| source | `builtin` |
| 文件 | [src/commands/keybindings/index.js](src/commands/keybindings/index.js) |

**说明**: 快捷键管理。

### /feedback

| 属性 | 值 |
|------|------|
| name | `feedback` |
| type | `prompt` |
| source | `builtin` |
| 文件 | [src/commands/feedback/index.js](src/commands/feedback/index.js) |

**说明**: 发送反馈。

### /btw

| 属性 | 值 |
|------|------|
| name | `btw` |
| type | `prompt` |
| source | `builtin` |
| 文件 | [src/commands/btw/index.js](src/commands/btw/index.js) |

**说明**: 快速笔记。

### /release-notes

| 属性 | 值 |
|------|------|
| name | `release-notes` |
| type | `local` |
| source | `builtin` |
| 文件 | [src/commands/release-notes/index.js](src/commands/release-notes/index.js) |

**说明**: 更新日志。

### /statusline

| 属性 | 值 |
|------|------|
| name | `statusline` |
| type | `local` |
| source | `builtin` |
| 文件 | [src/commands/statusline.js](src/commands/statusline.js) |

**说明**: 状态栏设置。

### /copy

| 属性 | 值 |
|------|------|
| name | `copy` |
| type | `local` |
| source | `builtin` |
| 文件 | [src/commands/copy/index.js](src/commands/copy/index.js) |

**说明**: 复制最后消息。

### /color

| 属性 | 值 |
|------|------|
| name | `color` |
| type | `local` |
| source | `builtin` |
| 文件 | [src/commands/color/index.js](src/commands/color/index.js) |

**说明**: 代理颜色设置。

### /stickers

| 属性 | 值 |
|------|------|
| name | `stickers` |
| type | `local-jsx` |
| source | `builtin` |
| 文件 | [src/commands/stickers/index.js](src/commands/stickers/index.js) |

**说明**: 贴纸管理。

### /mobile

| 属性 | 值 |
|------|------|
| name | `mobile` |
| type | `local` |
| source | `builtin` |
| 文件 | [src/commands/mobile/index.js](src/commands/mobile/index.js) |

**说明**: 移动端二维码。

### /exit

| 属性 | 值 |
|------|------|
| name | `exit` |
| type | `local` |
| source | `builtin` |
| 文件 | [src/commands/exit/index.js](src/commands/exit/index.js) |

**说明**: 退出 CLI。

### /advisor

| 属性 | 值 |
|------|------|
| name | `advisor` |
| type | `prompt` |
| source | `builtin` |
| 文件 | [src/commands/advisor.js](src/commands/advisor.js) |

**说明**: 顾问模型设置。

### /passes

| 属性 | 值 |
|------|------|
| name | `passes` |
| type | `local` |
| source | `builtin` |
| 文件 | [src/commands/passes/index.js](src/commands/passes/index.js) |

**说明**: Pass 管理。

### /sandbox-toggle

| 属性 | 值 |
|------|------|
| name | `sandbox-toggle` |
| type | `local` |
| source | `builtin` |
| 文件 | [src/commands/sandbox-toggle/index.js](src/commands/sandbox-toggle/index.js) |

**说明**: 沙箱模式切换。

## 条件命令

### Ant-only 命令

| 命令 | 说明 |
|------|------|
| `/commit` | Git 提交 |
| `/commit-push-pr` | 提交并推送 PR |
| `/autofix-pr` | 自动修复 PR |
| `/init-verifiers` | 初始化验证器 |
| `/issue` | 问题管理 |
| `/good-claude` | Good Claude |
| `/bughunter` | Bug Hunter |
| `/ctx_viz` | 上下文可视化 |
| `/backfill-sessions` | 回填会话 |
| `/break-cache` | 清除缓存 |
| `/mock-limits` | 模拟限制 |
| `/ultraplan` | Ultraplan |
| `/subscribe-pr` | 订阅 PR |
| `/ant-trace` | Ant 追踪 |
| `/perf-issue` | 性能问题 |
| `/env` | 环境变量 |
| `/oauth-refresh` | OAuth 刷新 |
| `/debug-tool-call` | 调试工具调用 |
| `/summary` | 会话摘要 |
| `/teleport` | 传送 |
| `/share` | 分享 |
| `/heapdump` | 堆转储 |

### Feature-gated 命令

| 命令 | Feature |
|------|---------|
| `/proactive` | `PROACTIVE` 或 `KAIROS` |
| `/brief` | `KAIROS` 或 `KAIROS_BRIEF` |
| `/assistant` | `KAIROS` |
| `/bridge` | `BRIDGE_MODE` |
| `/remote-control-server` | `DAEMON` && `BRIDGE_MODE` |
| `/voice` | `VOICE_MODE` |
| `/force-snip` | `HISTORY_SNIP` |
| `/workflows` | `WORKFLOW_SCRIPTS` |
| `/web` | `CCR_REMOTE_SETUP` |
| `/peers` | `UDS_INBOX` |
| `/fork` | `FORK_SUBAGENT` |
| `/buddy` | `BUDDY` |
| `/torch` | `TORCH` |

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
        if (!isClaudeAISubscriber() &&
            !isUsing3PServices() &&
            isFirstPartyAnthropicBaseUrl()) return true
        break
    }
  }
  return false
}
```

### 远程模式安全命令

```typescript
export const REMOTE_SAFE_COMMANDS: Set<Command> = new Set([
  session, exit, clear, help, theme, color, vim,
  cost, usage, copy, btw, feedback, plan,
  keybindings, statusline, stickers, mobile
])
```

### Bridge 安全命令

```typescript
export const BRIDGE_SAFE_COMMANDS: Set<Command> = new Set([
  compact, clear, cost, summary, releaseNotes, files
])

export function isBridgeSafeCommand(cmd: Command): boolean {
  if (cmd.type === 'local-jsx') return false
  if (cmd.type === 'prompt') return true
  return BRIDGE_SAFE_COMMANDS.has(cmd)
}
```

## 命令来源

| source | 说明 |
|--------|------|
| `builtin` | 内置命令 |
| `bundled` | 打包的技能 |
| `skills` | skills/ 目录中的技能 |
| `plugin` | 插件提供的命令 |
| `mcp` | MCP 服务器提供的命令 |
| `commands_DEPRECATED` | 旧版命令目录 |

## 关键文件

- [src/commands.ts](src/commands.ts) — 命令注册中心
- [src/types/command.ts](src/types/command.ts) — Command 类型定义
- [src/commands/](src/commands/) — 各命令实现目录
- [src/skills/](src/skills/) — 技能目录

## 关联文档

- [06-命令系统](06-command-system.md)
- [16-Skills系统](16-skills-system.md)