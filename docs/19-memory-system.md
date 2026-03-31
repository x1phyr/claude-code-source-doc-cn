# Memory 系统详解

## 概述

Memory 系统是 Claude Code 的持久化记忆机制，允许跨会话保存和检索信息。记忆存储在 `.claude/memory/` 目录中。

## Memory 类型

```typescript
type MemoryType =
  | 'user'       // 用户偏好和背景
  | 'feedback'   # 行为指导（避免/保持）
  | 'project'    // 项目上下文
  | 'reference'  // 外部资源引用
```

## 目录结构

```
.claude/memory/
├── MEMORY.md              # 索引文件
├── user_role.md           # 用户信息
├── feedback_style.md      # 反馈记忆
├── project_context.md     # 项目上下文
└── reference_docs.md      # 参考引用
```

## Memory 文件格式

### Frontmatter

```markdown
---
name: memory-name
description: 简短描述
type: user | feedback | project | reference
---

记忆内容...
```

### 结构化内容

对于 `feedback` 和 `project` 类型：

```markdown
---
name: testing-feedback
description: 测试相关反馈
type: feedback
---

不要在测试中使用 mock 数据库。

**Why:** 上季度发生过 mock 通过但生产迁移失败的事故。

**How to apply:** 集成测试必须连接真实数据库。
```

## MEMORY.md 索引

索引文件列出所有记忆：

```markdown
# Memory Index

- [User Role](user_role.md) — 用户背景信息
- [Testing Feedback](feedback_testing.md) — 测试相关反馈
- [API Reference](reference_api.md) — API 文档引用
```

### 索引限制

- 最多 200 行
- 每行不超过 150 字符
- 超出部分会被截断

## 记忆加载

### 加载流程

```
会话启动
    ↓
读取 MEMORY.md
    ↓
解析记忆文件
    ↓
注入到系统提示
    ↓
可用于当前会话
```

### 加载时机

记忆在会话启动时自动加载。不会在会话中途重新加载。

## 记忆创建

### 用户请求

用户可以请求保存记忆：

```
"Remember that I prefer TDD for new features"
```

### 自动提取

系统可以从会话中自动提取记忆：

```typescript
// src/services/extractMemories/extractMemories.ts
export async function extractMemories(
  messages: Message[]
): Promise<Memory[]> {
  // 分析会话内容
  // 识别值得记住的信息
  // 返回建议的记忆
}
```

## 记忆更新

### 更新规则

1. **检查重复**：先查找是否有现有记忆
2. **更新或创建**：更新现有记忆或创建新记忆
3. **更新索引**：同步更新 MEMORY.md

### 示例

```typescript
// 更新记忆
await updateMemory('testing-feedback', {
  content: '新的反馈内容...'
})

// 添加到索引
await addToIndex('testing-feedback.md', '测试反馈')
```

## 记忆删除

### 用户请求

```
"Forget the testing feedback"
```

### 删除流程

```typescript
export async function removeMemory(name: string): Promise<void> {
  // 删除记忆文件
  await fs.unlink(`.claude/memory/${name}.md`)

  // 从索引中移除
  await removeFromIndex(name)
}
```

## 记忆应用

### 系统提示注入

记忆内容被注入到系统提示中：

```typescript
export function buildSystemPrompt(memories: Memory[]): string {
  const memorySection = memories
    .map(m => `## ${m.name}\n\n${m.content}`)
    .join('\n\n')

  return `
...其他系统提示...

# Memories

${memorySection}

...其他系统提示...
`
}
```

### 上下文相关性

只有相关的记忆会被加载：

- 当前项目的记忆
- 用户级全局记忆
- 最近使用的记忆

## /memory 命令

### 功能

- 查看所有记忆
- 创建新记忆
- 更新现有记忆
- 删除记忆

### 使用

```
/memory                    # 打开记忆管理
/memory add <name>         # 添加记忆
/memory remove <name>      # 删除记忆
/memory refresh            # 重新加载记忆
```

## 记忆最佳实践

### 保持简洁

```markdown
---
name: code-style
description: 代码风格偏好
type: feedback
---

使用函数式风格，避免类。

**Why:** 项目历史原因，团队更熟悉函数式编程。
**How to apply:** 新代码使用纯函数和组合。
```

### 避免冗余

不要重复代码中已有的信息：
- 代码风格 → 使用 linter 配置
- 架构决策 → 使用 ADR 文档
- API 文档 → 使用 CLAUDE.md

### 及时更新

过时的记忆应该删除或更新：
- 项目变化时
- 偏好改变时
- 反馈不再适用时

## 关键文件

- [src/services/SessionMemory/](src/services/SessionMemory/) — 会话记忆服务
- [src/services/extractMemories/](src/services/extractMemories/) — 记忆提取
- [src/commands/memory/index.js](src/commands/memory/index.js) — /memory 命令

## 关联文档

- [20-CLAUDE.md系统](20-claudemd-system.md)
- [23-配置参考](23-configuration-reference.md)