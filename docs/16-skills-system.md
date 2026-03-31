# Skills 系统详解

## 概述

Skills（技能）是可扩展的提示模板，允许用户定义专门的能力。技能可以从多个来源加载：内置技能、技能目录、插件和 MCP 服务器。

## 技能类型

```typescript
type SkillSource =
  | 'bundled'      // 打包的内置技能
  | 'skills'       // skills/ 目录
  | 'plugin'       // 插件提供的技能
  | 'mcp'          // MCP 服务器提供的技能
```

## 技能加载

### 定义位置

[src/skills/](src/skills/)

```
skills/
├── bundledSkills.ts       # 内置技能注册
├── loadSkillsDir.ts       # 技能目录加载
├── skillDiscovery.ts      # 技能发现
└── skillUtils.ts          # 技能工具
```

### 加载流程

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
```

## 技能目录结构

```
skills/
├── my-skill/
│   ├── SKILL.md          # 技能定义（必需）
│   └── examples/
│       └── example.md    # 示例（可选）
```

## SKILL.md 格式

```markdown
---
name: my-skill
description: My skill description
whenToUse: When you need to do X
aliases:
  - ms
  - myskill
---

# My Skill

Detailed instructions for the skill...

## Usage

How to use this skill...

## Examples

Example usage...
```

### Frontmatter 字段

| 字段 | 必需 | 说明 |
|------|------|------|
| `name` | ✓ | 技能名称（用于 `/skill-name`） |
| `description` | ✓ | 简短描述 |
| `whenToUse` | | 使用时机提示 |
| `aliases` | | 别名列表 |

## 内置技能

### 定义位置

[src/skills/bundled/](src/skills/bundled/)

### 技能列表

| 技能 | 说明 |
|------|------|
| `claude-api` | Claude API 文档查询 |
| `claude-code-guide` | Claude Code 使用指南 |
| `verify` | 验证技能 |
| `keybindings` | 快捷键帮助 |
| `loop` | 循环执行 |
| `simplify` | 代码简化 |
| `remember` | 记忆管理 |
| `stuck` | 卡住时帮助 |
| `scheduleRemoteAgents` | 调度远程代理 |

### bundledSkills.ts

```typescript
export function getBundledSkills(): Command[] {
  return [
    claudeApiSkill,
    claudeCodeGuideSkill,
    verifySkill,
    keybindingsSkill,
    loopSkill,
    simplifySkill,
    rememberSkill,
    stuckSkill,
    scheduleRemoteAgentsSkill,
  ]
}
```

## 技能命令

技能被注册为 `prompt` 类型的命令：

```typescript
type SkillCommand = {
  type: 'prompt'
  name: string                  // 技能名称
  description: string           // 描述
  source: 'bundled' | 'skills' | 'plugin' | 'mcp'
  getPromptForCommand(args, context): Promise<string>
}
```

## 动态技能发现

### 基于文件操作发现

```typescript
// 在文件操作中发现技能目录
export function discoverSkillDirsForPaths(paths: string[]): string[]

// 激活条件技能
export function activateConditionalSkillsForPaths(paths: string[]): void
```

### 动态技能注册

```typescript
// 添加技能目录
export function addSkillDirectories(dirs: string[]): void

// 获取动态技能
export function getDynamicSkills(): Command[]
```

## 技能调用

### 通过 SkillTool

```typescript
const result = await SkillTool.call({
  skill: 'my-skill',
  args: '--option value'
}, context)
```

### 通过斜杠命令

```
/my-skill arg1 arg2
```

## 技能搜索

### 目录结构

[src/services/skillSearch/](src/services/skillSearch/)

```
skillSearch/
├── localSearch.ts         # 本地搜索
├── indexBuilder.ts        # 索引构建
└── searchUtils.ts         # 搜索工具
```

### 搜索功能

```typescript
// 构建技能索引
export function buildSkillIndex(skills: Command[]): SkillIndex

// 搜索技能
export function searchSkills(
  query: string,
  index: SkillIndex
): Command[]
```

## 技能缓存

```typescript
// 清除技能缓存
export function clearSkillCaches(): void

// 清除命令缓存
export function clearCommandMemoizationCaches(): void
```

## 插件技能

### 加载插件技能

```typescript
// 从插件加载技能
export async function getPluginSkills(): Promise<Command[]>
```

### 内置插件技能

```typescript
// 获取内置插件技能
export function getBuiltinPluginSkillCommands(): Command[]
```

## MCP 技能

### 过滤 MCP 技能

```typescript
export function getMcpSkillCommands(
  mcpCommands: readonly Command[]
): readonly Command[] {
  return mcpCommands.filter(
    cmd =>
      cmd.type === 'prompt' &&
      cmd.loadedFrom === 'mcp' &&
      !cmd.disableModelInvocation
  )
}
```

## 技能与命令的关系

技能是一种特殊的命令：

```
Command
├── prompt 类型
│   ├── Skill（技能）
│   │   ├── bundled
│   │   ├── skills/
│   │   ├── plugin
│   │   └── mcp
│   └── 其他 prompt 命令
├── local 类型
└── local-jsx 类型
```

## 技能最佳实践

### 描述清晰

```markdown
---
name: test-runner
description: Run tests with coverage
whenToUse: When you need to run tests or check test coverage
---
```

### 提供使用示例

```markdown
## Usage

/test-runner [--coverage] [--watch] [test-file]

## Examples

/test-runner --coverage
/test-runner src/utils/__tests__/format.test.ts
```

### 结构化内容

```markdown
# Test Runner

## Overview
Brief description...

## Usage
How to invoke...

## Options
- `--coverage`: Generate coverage report
- `--watch`: Watch mode

## Examples
Code examples...
```

## 关键文件

- [src/skills/bundledSkills.ts](src/skills/bundledSkills.ts) — 内置技能注册
- [src/skills/loadSkillsDir.ts](src/skills/loadSkillsDir.ts) — 技能目录加载
- [src/commands.ts](src/commands.ts) — 命令注册中心
- [src/tools/SkillTool/SkillTool.ts](src/tools/SkillTool/SkillTool.ts) — 技能工具

## 关联文档

- [06-命令系统](06-command-system.md)
- [11-命令清单](11-commands-catalog.md)
- [08-MCP集成](08-mcp-integration.md)