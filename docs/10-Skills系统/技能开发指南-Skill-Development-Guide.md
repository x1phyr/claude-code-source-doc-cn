# 技能开发指南

## 概述

Skills（技能）是 Claude Code 的可扩展机制，允许用户创建专门领域的自定义能力。本指南介绍如何开发、测试和发布自定义技能。

## 核心文件

- [src/skills/loadSkillsDir.ts](src/skills/loadSkillsDir.ts) — 技能加载
- [src/skills/bundledSkills.ts](src/skills/bundledSkills.ts) — 内置技能
- [src/tools/SkillTool/SkillTool.ts](src/tools/SkillTool/SkillTool.ts) — 技能工具

## 技能结构

### 目录结构

```
.claude/skills/
├── my-skill/
│   ├── SKILL.md           # 必需：技能定义
│   ├── README.md          # 可选：详细文档
│   └── examples/          # 可选：示例文件
│       └── example.md
```

### SKILL.md 格式

```markdown
---
name: my-skill
description: My custom skill for specific tasks
userInvocable: true
---

# My Skill

Instructions for the skill...

## When to use

- Use case 1
- Use case 2

## How to use

Step-by-step instructions...

## Examples

### Example 1

Input: ...
Output: ...
```

## Frontmatter 字段

### 必需字段

```yaml
name: string           # 技能名称（kebab-case）
description: string    # 简短描述（显示在技能列表中）
```

### 可选字段

```yaml
userInvocable: boolean       # 用户可调用（默认 false）
model: string                # 指定模型（haiku/sonnet/opus）
tools: string[]              # 工具白名单
maxTurns: number             # 最大轮次
isConcurrencySafe: boolean   # 并发安全
```

## 技能开发

### 创建基本技能

```markdown
---
name: code-review
description: Review code changes for quality and security
userInvocable: true
---

# Code Review Skill

You are a code reviewer. Analyze the provided code changes and provide feedback.

## Review Checklist

### Code Quality
- [ ] Code is readable and well-organized
- [ ] Functions are focused and single-purpose
- [ ] Variable names are descriptive
- [ ] Comments explain "why", not "what"

### Security
- [ ] No hardcoded secrets or credentials
- [ ] Input validation is present
- [ ] No SQL injection vulnerabilities
- [ ] No XSS vulnerabilities

### Performance
- [ ] No obvious performance issues
- [ ] Appropriate data structures used
- [ ] No unnecessary loops or recursion

## Output Format

Provide your review in the following format:

### Summary
Brief overall assessment

### Issues
List of issues found, categorized by severity:
- 🔴 Critical: ...
- 🟡 Warning: ...
- 🟢 Suggestion: ...

### Recommendations
Specific recommendations for improvement
```

### 添加工具限制

```yaml
---
name: safe-analysis
description: Analyze code without modifications
tools:
  - Read
  - Glob
  - Grep
---
```

### 指定模型

```yaml
---
name: quick-task
description: Fast task for simple operations
model: haiku
---
```

## 技能加载

### 加载流程

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

export async function loadSkill(path: string): Promise<LoadedSkill> {
  const content = await readFile(path, 'utf-8')
  const { data, content: body } = parseFrontmatter(content)

  // 提取 When to use 部分
  const whenToUseMatch = body.match(/## When to use\n([\s\S]*?)(?=\n##|$)/)

  return {
    name: data.name ?? basename(dirname(path)),
    description: data.description ?? '',
    content: body,
    path,
    whenToUse: whenToUseMatch?.[1]?.trim(),
    userInvocable: data.userInvocable ?? false,
    model: data.model,
    tools: data.tools,
  }
}
```

### 加载技能接口

```typescript
export interface LoadedSkill {
  name: string
  description: string
  content: string
  path: string
  whenToUse?: string
  userInvocable?: boolean
  model?: string
  tools?: string[]
  maxTurns?: number
  isConcurrencySafe?: boolean
}
```

## 技能调用

### SkillTool 实现

```typescript
export class SkillTool implements Tool {
  name = 'Skill'

  async call(
    args: { skill: string; args?: string },
    context: ToolUseContext,
  ): Promise<ToolResult> {
    // 1. 解析技能名称
    const skillName = parseSkillName(args.skill)
    const skill = await loadSkillByName(skillName, context)

    if (!skill) {
      throw new Error(`Skill not found: ${skillName}`)
    }

    // 2. 格式化技能提示
    const skillPrompt = formatSkillPrompt(skill, args.args)

    // 3. 创建技能上下文
    const skillContext = createSkillContext(skill, context)

    // 4. 执行技能
    const result = await executeSkill(skill, skillPrompt, skillContext)

    return {
      output: {
        type: 'skill_result',
        skill: skill.name,
        result,
      },
    }
  }
}
```

### 格式化技能提示

```typescript
function formatSkillPrompt(
  skill: LoadedSkill,
  args?: string,
): string {
  let prompt = skill.content

  // 替换参数占位符
  if (args) {
    prompt = prompt.replace('$ARGUMENTS', args)
  }

  return prompt
}
```

## 内置技能

### 查看内置技能列表

```typescript
export const BUNDLED_SKILLS = [
  'init',
  'git-bisect',
  'pr-description',
  'review-pr',
  'summarize-todos',
  // ...
]

export function isBundledSkill(name: string): boolean {
  return BUNDLED_SKILLS.includes(name)
}
```

### 内置技能示例

```typescript
// src/skills/bundled/init/SKILL.md
export const INIT_SKILL = {
  name: 'init',
  description: 'Initialize CLAUDE.md for a new project',
  userInvocable: true,
  content: `
# Init Skill

Analyze this codebase and create a CLAUDE.md file...

## What to add
1. Build, lint, test commands
2. Architecture overview
3. Key files and patterns
...
  `,
}
```

## 用户调用技能

### 技能发现

```typescript
// 用户可调用技能列表
export function getUserInvocableSkills(
  skills: Map<string, LoadedSkill>,
): LoadedSkill[] {
  return [...skills.values()].filter(s => s.userInvocable)
}
```

### 命令行调用

```bash
# 调用技能
claude /skill my-skill

# 带参数调用
claude /skill my-skill --arg "some argument"
```

## 技能最佳实践

### 1. 清晰的触发条件

```markdown
## When to use

Use this skill when:
- User asks for a code review
- User mentions "review this code"
- User provides a diff or PR link
```

### 2. 结构化输出

```markdown
## Output Format

Always structure your output as follows:

### Summary
One paragraph summary of findings

### Details
Detailed analysis organized by category

### Action Items
- [ ] Item 1
- [ ] Item 2
```

### 3. 示例驱动

```markdown
## Examples

### Example 1: Simple Function Review

**Input:**
\`\`\`javascript
function add(a, b) {
  return a + b
}
\`\`\`

**Output:**
### Summary
Simple addition function with no issues.

### Details
- Well-named parameters
- Clear single responsibility
- No security concerns

### Action Items
None required.
```

### 4. 错误处理

```markdown
## Error Handling

If you encounter:
- Missing files: Report which files are missing
- Parse errors: Show the specific error and line number
- Permission issues: Suggest how to resolve

Always provide actionable guidance for errors.
```

## 测试技能

### 手动测试

```bash
# 加载技能
claude

# 调用技能
> /skill my-skill

# 检查输出
```

### 单元测试

```typescript
describe('my-skill', () => {
  it('should load skill correctly', async () => {
    const skill = await loadSkill('.claude/skills/my-skill/SKILL.md')
    expect(skill.name).toBe('my-skill')
    expect(skill.userInvocable).toBe(true)
  })

  it('should format prompt with arguments', () => {
    const prompt = formatSkillPrompt(skill, 'test-args')
    expect(prompt).toContain('test-args')
  })
})
```

## 发布技能

### 项目级技能

```
your-project/
├── .claude/
│   └── skills/
│       └── project-skill/
│           └── SKILL.md
```

### 共享技能包

```typescript
// 插件 manifest 中包含技能
{
  "name": "my-skill-pack",
  "version": "1.0.0",
  "skills": [
    {
      "name": "skill-1",
      "path": "skills/skill-1/SKILL.md"
    },
    {
      "name": "skill-2",
      "path": "skills/skill-2/SKILL.md"
    }
  ]
}
```

## 调试技能

### 启用调试模式

```typescript
// 在 SKILL.md 中添加调试信息
## Debug Mode

When environment variable DEBUG_SKILL=1, output:
- Skill execution time
- Token usage
- Intermediate steps
```

### 日志输出

```typescript
// 技能执行日志
logEvent('tengu_skill_invoked', {
  skill_name: skill.name,
  has_args: !!args,
  model: skill.model,
})

logEvent('tengu_skill_completed', {
  skill_name: skill.name,
  duration_ms: endTime - startTime,
  tokens_used: totalTokens,
})
```

## 技能限制

### 工具限制

```yaml
# 只读技能
tools:
  - Read
  - Glob
  - Grep

# 完整访问
tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
  - Grep
```

### 模型限制

```yaml
# 使用快速模型
model: haiku

# 使用高级模型
model: opus
```

### 轮次限制

```yaml
# 单轮完成
maxTurns: 1

# 多轮交互
maxTurns: 5
```

## 关键文件

- [src/skills/loadSkillsDir.ts](src/skills/loadSkillsDir.ts) — 技能加载
- [src/tools/SkillTool/SkillTool.ts](src/tools/SkillTool/SkillTool.ts) — 技能工具
- [src/skills/bundledSkills.ts](src/skills/bundledSkills.ts) — 内置技能注册

## 关联文档

- [10-Skills系统/Skills系统详解-Skills-System.md](Skills系统详解-Skills-System.md)
- [11-插件系统/插件系统-Plugin-System.md](../11-插件系统/插件系统-Plugin-System.md)
- [04-命令系统/命令系统详解-Command-System.md](../04-命令系统/命令系统详解-Command-System.md)