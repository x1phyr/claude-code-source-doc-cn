# 插件系统

## 概述

插件系统允许第三方扩展 Claude Code 的功能，包括自定义命令、工具和 Hook 集成。

## 核心文件

- [src/services/mcp/config.ts](src/services/mcp/config.ts) — 配置加载
- [src/skills/](src/skills/) — 技能加载
- [src/commands.ts](src/commands.ts) — 命令注册

## 插件类型

### MCP 服务器

通过 Model Context Protocol 提供工具和资源：

```json
{
  "mcpServers": {
    "my-server": {
      "command": "npx",
      "args": ["-y", "my-mcp-server"],
      "env": {
        "API_KEY": "${MY_API_KEY}"
      }
    }
  }
}
```

### 技能 (Skills)

提供专门的领域能力：

```
.claude/skills/
├── my-skill/
│   └── SKILL.md
```

### 命令

自定义斜杠命令：

```
.claude/commands/
├── my-command.md
```

## MCP 插件

### 服务器发现

```typescript
export function getAllMcpConfigs(): Record<string, McpServerConfig> {
  const configs: Record<string, McpServerConfig> = {}

  // 从设置文件加载
  for (const source of SETTING_SOURCES) {
    const settings = getSettingsForSource(source)
    if (settings?.mcpServers) {
      for (const [name, config] of Object.entries(settings.mcpServers)) {
        if (!isMcpServerDisabled(name, source)) {
          configs[name] = config
        }
      }
    }
  }

  return configs
}
```

### 项目级 MCP

```typescript
// .mcp.json 格式
{
  "mcpServers": {
    "project-server": {
      "type": "stdio",
      "command": "./scripts/mcp-server.sh"
    }
  }
}

// 加载项目 MCP
export async function loadProjectMcpConfig(cwd: string): Promise<void> {
  const mcpJsonPath = join(cwd, '.mcp.json')
  if (existsSync(mcpJsonPath)) {
    const content = await readFile(mcpJsonPath, 'utf-8')
    const config = JSON.parse(content)
    // 合并到设置中
    mergeMcpConfig(config)
  }
}
```

## 技能插件

### SKILL.md 格式

```markdown
---
name: my-skill
description: My custom skill for specific tasks
---

# My Skill

Instructions for the skill...

## When to use

- Use case 1
- Use case 2

## How to use

Step-by-step instructions...
```

### 技能加载

```typescript
export async function loadSkillsFromDirectory(
  skillsDir: string,
): Promise<Map<string, LoadedSkill>> {
  const skills = new Map<string, LoadedSkill>()

  const entries = await readdir(skillsDir, { withFileTypes: true })
  for (const entry of entries) {
    if (entry.isDirectory()) {
      const skillPath = join(skillsDir, entry.name, 'SKILL.md')
      if (existsSync(skillPath)) {
        const skill = await loadSkill(skillPath)
        skills.set(skill.name, skill)
      }
    }
  }

  return skills
}

export interface LoadedSkill {
  name: string
  description: string
  content: string
  path: string
  whenToUse?: string
}
```

### 技能调用

```typescript
export class SkillTool implements Tool {
  name = 'Skill'

  async call(args: { skill: string }, context: ToolUseContext): Promise<ToolResult> {
    // 加载技能
    const skill = await loadSkill(args.skill)
    if (!skill) {
      throw new Error(`Skill not found: ${args.skill}`)
    }

    // 注入技能内容作为系统提示
    const skillPrompt = formatSkillPrompt(skill)

    return {
      output: {
        type: 'skill_loaded',
        skill: skill.name,
        prompt: skillPrompt,
      },
    }
  }
}
```

## 命令插件

### 命令文件格式

```markdown
---
name: my-command
description: My custom command
---

Command instructions and prompt content...

$ARGUMENTS will be replaced with user input.
```

### 命令发现

```typescript
export async function discoverCommands(
  commandsDir: string,
): Promise<Map<string, Command>> {
  const commands = new Map<string, Command>()

  const files = await glob('*.md', { cwd: commandsDir })
  for (const file of files) {
    const command = await loadCommand(join(commandsDir, file))
    if (command) {
      commands.set(command.name, command)
    }
  }

  return commands
}

export async function loadCommand(path: string): Promise<Command | null> {
  const content = await readFile(path, 'utf-8')
  const { data, content: body } = parseFrontmatter(content)

  return {
    type: 'prompt',
    name: data.name ?? basename(path, '.md'),
    description: data.description ?? '',
    source: 'plugin',
    loadedFrom: path,
    getPromptForCommand: async (args) => {
      return body.replace('$ARGUMENTS', args ?? '')
    },
  }
}
```

## Hook 集成

### 插件定义 Hook

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": ["my-validator.sh"]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": ["my-formatter.sh"]
      }
    ]
  }
}
```

### Hook 执行

```typescript
export async function executeHook(
  hook: string,
  context: HookContext,
): Promise<HookResult> {
  const result = await execFile(hook, {
    env: {
      CLAUDE_TOOL_NAME: context.toolName,
      CLAUDE_TOOL_INPUT: JSON.stringify(context.input),
      CLAUDE_SESSION_ID: context.sessionId,
      CLAUDE_CWD: context.cwd,
    },
  })

  return {
    stdout: result.stdout,
    stderr: result.stderr,
    exitCode: result.exitCode,
  }
}
```

## 安全考虑

### 插件权限

```typescript
// 项目级 MCP 需要用户批准
export async function approveProjectMcp(
  serverName: string,
  config: McpServerConfig,
): Promise<boolean> {
  const approval = await askUserPermission({
    type: 'mcp_server',
    message: `Allow MCP server "${serverName}" from project?`,
    detail: JSON.stringify(config, null, 2),
  })

  return approval === 'allow'
}
```

### 沙箱隔离

```typescript
// 插件在沙箱中执行
export async function runPluginInSandbox(
  plugin: Plugin,
  input: unknown,
): Promise<unknown> {
  if (SandboxManager.isSandboxingEnabled()) {
    const wrappedCommand = await SandboxManager.wrapWithSandbox(
      plugin.command,
      plugin.shell,
    )
    // 执行包装后的命令
  }
}
```

## 插件市场

### 发现插件

```typescript
export interface PluginManifest {
  name: string
  version: string
  description: string
  author: string
  repository?: string
  mcpServers?: Record<string, McpServerConfig>
  skills?: SkillDefinition[]
  commands?: CommandDefinition[]
}

export async function fetchPluginManifest(
  pluginUrl: string,
): Promise<PluginManifest> {
  const response = await fetch(pluginUrl)
  return response.json()
}
```

### 安装插件

```typescript
export async function installPlugin(
  manifest: PluginManifest,
): Promise<void> {
  // 添加 MCP 服务器到设置
  if (manifest.mcpServers) {
    for (const [name, config] of Object.entries(manifest.mcpServers)) {
      await addMcpServer(name, config)
    }
  }

  // 复制技能文件
  if (manifest.skills) {
    for (const skill of manifest.skills) {
      await installSkill(skill)
    }
  }

  // 复制命令文件
  if (manifest.commands) {
    for (const command of manifest.commands) {
      await installCommand(command)
    }
  }
}
```

## 关键文件

- [src/services/mcp/config.ts](src/services/mcp/config.ts) — MCP 配置
- [src/skills/loadSkillsDir.ts](src/skills/loadSkillsDir.ts) — 技能加载
- [src/commands.ts](src/commands.ts) — 命令注册
- [src/utils/hooks.ts](src/utils/hooks.ts) — Hook 系统

## 关联文档

- [11-插件系统/Hook系统详解-Hooks-System.md](Hook系统详解-Hooks-System.md)
- [06-MCP协议/MCP集成详解-MCP-Integration.md](../06-MCP协议/MCP集成详解-MCP-Integration.md)
- [10-Skills系统/Skills系统详解-Skills-System.md](../10-Skills系统/Skills系统详解-Skills-System.md)