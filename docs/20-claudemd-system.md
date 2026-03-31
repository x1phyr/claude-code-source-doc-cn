# CLAUDE.md 系统详解

## 概述

CLAUDE.md 是项目级指令文件，为 Claude 提供项目上下文、编码规范和开发指南。它在会话启动时被自动发现并注入到系统提示中。

## 文件发现

### 发现机制

系统从多个目录发现 CLAUDE.md 文件：

```typescript
export function discoverClaudeMdFiles(cwd: string): string[] {
  const files = []

  // 向上遍历目录树
  let dir = cwd
  while (true) {
    const candidate = path.join(dir, 'CLAUDE.md')
    if (fs.existsSync(candidate)) {
      files.push(candidate)
    }

    const parent = path.dirname(dir)
    if (parent === dir) break  // 到达根目录
    dir = parent
  }

  return files.reverse()  // 从根到当前目录
}
```

### 发现顺序

```
/CLAUDE.md                    # 根目录
/src/CLAUDE.md                # src 目录
/src/components/CLAUDE.md     # 组件目录
```

文件从根目录到当前目录依次加载，后面的文件可以覆盖前面的设置。

## 文件格式

### 基本结构

```markdown
# 项目名称

简短描述...

## 项目概况

- 技术栈
- 架构决策
- 关键约束

## 开发命令

\`\`\`bash
bun run dev      # 开发服务器
bun run test     # 运行测试
bun run build    # 构建
\`\`\`

## 代码规范

- 使用 TypeScript
- 函数式风格优先
- 测试驱动开发

## 重要注意事项

- 不要修改 X 模块
- Y 功能需要特殊处理
```

### 推荐内容

1. **项目背景**：简要说明项目目的
2. **技术栈**：使用的主要技术
3. **常用命令**：开发、测试、构建命令
4. **架构决策**：重要的架构选择
5. **编码规范**：代码风格和要求
6. **注意事项**：需要特别关注的事项

## 系统提示注入

### 注入流程

```
发现 CLAUDE.md 文件
    ↓
读取并解析内容
    ↓
处理外部引用
    ↓
构建系统提示
    ↓
注入到 Claude 上下文
```

### 注入位置

CLAUDE.md 内容被注入到系统提示的特定位置：

```typescript
export function buildSystemPrompt(
  basePrompt: string,
  claudeMdContent: string
): string {
  return `
${basePrompt}

# Project Context

${claudeMdContent}
`
}
```

## 外部引用

### 语法

```markdown
@file:path/to/file.md
```

### 解析

```typescript
export function resolveExternalReferences(
  content: string,
  baseDir: string
): string {
  return content.replace(/@file:(.+)/g, (_, refPath) => {
    const fullPath = path.resolve(baseDir, refPath)
    return fs.readFileSync(fullPath, 'utf-8')
  })
}
```

### 示例

```markdown
# 项目配置

参见 @file:docs/architecture.md 了解架构设计。

测试指南：@file:docs/testing-guide.md
```

## CLAUDE.md 与 Memory 的区别

| 特性 | CLAUDE.md | Memory |
|------|-----------|--------|
| 作用域 | 项目级 | 用户/项目级 |
| 版本控制 | 可提交到 Git | 通常不提交 |
| 内容类型 | 项目配置和规范 | 偏好和反馈 |
| 更新频率 | 项目变更时 | 交互中动态更新 |
| 发现机制 | 自动发现 | 手动管理 |

## 最佳实践

### 保持简洁

```markdown
# 我的项目

React + TypeScript 前端项目。

## 命令

- `npm run dev` — 开发服务器
- `npm run test` — 运行测试

## 规范

- 使用函数组件和 Hooks
- 测试覆盖新功能
```

### 不要重复

避免重复已有信息：
- 代码规范 → 使用 ESLint/Prettier 配置
- API 文档 → 使用专门的文档文件
- 类型定义 → 使用 TypeScript

### 使用外部引用

大型文档应该引用外部文件：

```markdown
# 项目文档

架构设计：@file:docs/architecture.md
API 文档：@file:docs/api.md
```

### 更新及时

项目变化时更新 CLAUDE.md：
- 新增主要依赖
- 架构重构
- 新的开发流程

## /init 命令

### 功能

自动生成初始 CLAUDE.md 文件。

### 执行

```
/init
```

### 生成内容

```typescript
export async function generateClaudeMd(cwd: string): Promise<string> {
  const packageJson = await readPackageJson(cwd)
  const techStack = detectTechStack(cwd)

  return `# ${packageJson.name}

${packageJson.description || '项目描述'}

## 技术栈

${techStack.map(t => `- ${t}`).join('\n')}

## 开发命令

\`\`\`bash
npm run dev      # 开发服务器
npm run test     # 运行测试
npm run build    # 构建
\`\`\`
`
}
```

## 关键文件

- [src/utils/claudeMd.ts](src/utils/claudeMd.ts) — CLAUDE.md 处理
- [src/commands/init.js](src/commands/init.js) — /init 命令
- [src/utils/systemPrompt.ts](src/utils/systemPrompt.ts) — 系统提示构建

## 关联文档

- [19-Memory系统](19-memory-system.md)
- [03-启动流程](03-entry-and-bootstrap.md)