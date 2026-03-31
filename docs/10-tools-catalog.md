# 工具清单

## 概述

本文档详细列出 Claude Code 所有内置工具的定义、属性和行为特征。

## 工具注册

### 定义位置

[src/tools.ts](src/tools.ts)

### 工具获取流程

```typescript
export function getAllBaseTools(): Tools {
  return [
    AgentTool,
    TaskOutputTool,
    BashTool,
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
    ExitPlanModeV2Tool,
    FileReadTool,
    FileEditTool,
    FileWriteTool,
    NotebookEditTool,
    WebFetchTool,
    TodoWriteTool,
    WebSearchTool,
    TaskStopTool,
    AskUserQuestionTool,
    SkillTool,
    EnterPlanModeTool,
    // 条件性加载的工具...
  ]
}

export function getTools(permissionContext: ToolPermissionContext): Tools {
  // 1. 简单模式过滤
  if (isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)) {
    return [BashTool, FileReadTool, FileEditTool]
  }

  // 2. 获取所有基础工具
  const tools = getAllBaseTools()

  // 3. 按 deny 规则过滤
  let allowedTools = filterToolsByDenyRules(tools, permissionContext)

  // 4. REPL 模式过滤
  if (isReplModeEnabled()) {
    allowedTools = allowedTools.filter(t => !REPL_ONLY_TOOLS.has(t.name))
  }

  // 5. isEnabled 检查
  return allowedTools.filter(t => t.isEnabled())
}
```

## 文件操作工具

### FileReadTool

**文件**: [src/tools/FileReadTool/FileReadTool.ts](src/tools/FileReadTool/FileReadTool.ts)

| 属性 | 值 |
|------|------|
| name | `Read` |
| 别名 | `FileRead` |
| 只读 | ✓ true |
| 破坏性 | ✗ false |
| 并发安全 | ✓ true |

**输入 Schema**:
```typescript
z.object({
  file_path: z.string().describe("绝对路径"),
  limit: z.number().optional().describe("读取行数限制"),
  offset: z.number().optional().describe("起始行偏移"),
  pages: z.string().optional().describe("PDF 页面范围 (如 '1-5')")
})
```

**特殊功能**:
- 支持图片读取（自动检测格式）
- 支持 PDF 文件读取（可指定页面范围）
- 支持 Jupyter Notebook (.ipynb)
- 自动检测二进制文件并阻止
- 文件不存在时提供相似文件建议

### FileEditTool

**文件**: [src/tools/FileEditTool/FileEditTool.ts](src/tools/FileEditTool/FileEditTool.ts)

| 属性 | 值 |
|------|------|
| name | `Edit` |
| 别名 | `FileEdit` |
| 只读 | ✗ false |
| 破坏性 | ✗ false |
| 并发安全 | 部分（同一文件不安全） |

**输入 Schema**:
```typescript
z.object({
  file_path: z.string().describe("绝对路径"),
  old_string: z.string().describe("要替换的文本（必须唯一）"),
  new_string: z.string().describe("替换后的文本"),
  replace_all: z.boolean().optional().describe("替换所有匹配")
})
```

**特殊功能**:
- 必须先读取文件才能编辑
- 精确字符串匹配（保留缩进）
- 支持全局替换 (`replace_all`)
- 编辑前验证 old_string 唯一性

### FileWriteTool

**文件**: [src/tools/FileWriteTool/FileWriteTool.ts](src/tools/FileWriteTool/FileWriteTool.ts)

| 属性 | 值 |
|------|------|
| name | `Write` |
| 别名 | `FileWrite` |
| 只读 | ✗ false |
| 破坏性 | ✓ true |
| 并发安全 | ✗ false |

**输入 Schema**:
```typescript
z.object({
  file_path: z.string().describe("绝对路径"),
  content: z.string().describe("文件内容")
})
```

**特殊功能**:
- 覆盖现有文件（危险操作）
- 必须先读取现有文件
- 自动创建目录

### GlobTool

**文件**: [src/tools/GlobTool/GlobTool.ts](src/tools/GlobTool/GlobTool.ts)

| 属性 | 值 |
|------|------|
| name | `Glob` |
| 只读 | ✓ true |
| 破坏性 | ✗ false |
| 并发安全 | ✓ true |

**输入 Schema**:
```typescript
z.object({
  pattern: z.string().describe("Glob 模式（如 '**/*.ts'）"),
  path: z.string().optional().describe("搜索目录")
})
```

**特殊功能**:
- 按修改时间排序结果
- 快速文件模式匹配
- 支持标准 glob 语法

### GrepTool

**文件**: [src/tools/GrepTool/GrepTool.ts](src/tools/GrepTool/GrepTool.ts)

| 属性 | 值 |
|------|------|
| name | `Grep` |
| 只读 | ✓ true |
| 破坏性 | ✗ false |
| 并发安全 | ✓ true |

**输入 Schema**:
```typescript
z.object({
  pattern: z.string().describe("正则表达式"),
  path: z.string().optional().describe("搜索路径"),
  output_mode: z.enum(['content', 'files_with_matches', 'count']).optional(),
  glob: z.string().optional().describe("文件过滤模式"),
  type: z.string().optional().describe("文件类型（js, py, rust 等）"),
  i: z.boolean().optional().describe("忽略大小写"),
  head_limit: z.number().optional().describe("结果数量限制")
})
```

**特殊功能**:
- 支持完整正则语法
- 多种输出模式
- 文件类型过滤
- 大结果集自动截断

## 执行工具

### BashTool

**文件**: [src/tools/BashTool/BashTool.tsx](src/tools/BashTool/BashTool.tsx)

| 属性 | 值 |
|------|------|
| name | `Bash` |
| 只读 | ✗ false（取决于命令） |
| 破坏性 | ✗ false（取决于命令） |
| 并发安全 | 部分 |

**输入 Schema**:
```typescript
z.object({
  command: z.string().describe("Shell 命令"),
  description: z.string().describe("命令描述"),
  timeout: z.number().optional().describe("超时（毫秒）"),
  run_in_background: z.boolean().optional().describe("后台运行")
})
```

**特殊功能**:
- 自动检测危险命令
- 沙箱模式支持
- 支持后台执行
- 自动超时管理
- 搜索/读取命令可折叠显示
- Git 操作特殊处理

**命令分类**:
| 类型 | 命令 |
|------|------|
| 搜索 | `find`, `grep`, `rg`, `ag`, `ack`, `locate` |
| 读取 | `cat`, `head`, `tail`, `less`, `more`, `wc`, `stat` |
| 列表 | `ls`, `tree`, `du` |
| 静默 | `mv`, `cp`, `rm`, `mkdir`, `chmod`, `touch` |

### PowerShellTool

**文件**: [src/tools/PowerShellTool/PowerShellTool.ts](src/tools/PowerShellTool/PowerShellTool.ts)

| 属性 | 值 |
|------|------|
| name | `PowerShell` |
| 启用条件 | Windows 平台或 `POWERSHELL_ENABLED=true` |

**输入 Schema**:
```typescript
z.object({
  command: z.string().describe("PowerShell 命令"),
  description: z.string().describe("命令描述")
})
```

## 任务管理工具

### AgentTool

**文件**: [src/tools/AgentTool/AgentTool.tsx](src/tools/AgentTool/AgentTool.tsx)

| 属性 | 值 |
|------|------|
| name | `Agent` |
| 别名 | `Task` |
| 并发安全 | ✓ true |

**输入 Schema**:
```typescript
z.object({
  description: z.string().describe("任务描述（3-5 词）"),
  prompt: z.string().describe("任务详细提示"),
  subagent_type: z.string().optional().describe("专用代理类型"),
  model: z.enum(['sonnet', 'opus', 'haiku']).optional(),
  run_in_background: z.boolean().optional().describe("后台运行"),
  name: z.string().optional().describe("代理名称（用于 SendMessage）"),
  isolation: z.enum(['worktree', 'remote']).optional(),
  cwd: z.string().optional()
})
```

**内置代理类型**:
| 类型 | 说明 |
|------|------|
| `general-purpose` | 通用代理 |
| `Explore` | 快速探索代码库 |
| `Plan` | 架构规划代理 |
| `code-reviewer` | 代码审查代理 |
| `claude-code-guide` | Claude Code 使用指南 |
| `statusline-setup` | 状态栏配置 |
| `verification` | 验证代理 |

**特殊功能**:
- 支持后台异步执行
- 支持工作树隔离模式
- 支持远程代理
- 代理名称注册（用于 SendMessage）
- 多代理/团队支持

### TaskOutputTool

**文件**: [src/tools/TaskOutputTool/TaskOutputTool.ts](src/tools/TaskOutputTool/TaskOutputTool.ts)

| 属性 | 值 |
|------|------|
| name | `TaskOutput` |
| 只读 | ✓ true |

**输入 Schema**:
```typescript
z.object({
  task_id: z.string().describe("任务 ID"),
  block: z.boolean().optional().describe("是否等待完成"),
  timeout: z.number().optional().describe("等待超时")
})
```

### TaskStopTool

**文件**: [src/tools/TaskStopTool/TaskStopTool.ts](src/tools/TaskStopTool/TaskStopTool.ts)

| 属性 | 值 |
|------|------|
| name | `TaskStop` |
| 破坏性 | ✓ true |

**输入 Schema**:
```typescript
z.object({
  task_id: z.string().describe("任务 ID"),
  shell_id: z.string().optional() // 已废弃
})
```

### TodoWriteTool

**文件**: [src/tools/TodoWriteTool/TodoWriteTool.ts](src/tools/TodoWriteTool/TodoWriteTool.ts)

| 属性 | 值 |
|------|------|
| name | `TodoWrite` |
| 只读 | ✗ false |

**输入 Schema**:
```typescript
z.object({
  todos: z.array(z.object({
    content: z.string().describe("任务内容"),
    status: z.enum(['pending', 'in_progress', 'completed']),
    activeForm: z.string().describe("进行时形式")
  }))
})
```

### Task 系列工具（Todo V2）

| 工具 | 说明 |
|------|------|
| TaskCreateTool | 创建任务 |
| TaskGetTool | 获取任务 |
| TaskUpdateTool | 更新任务 |
| TaskListTool | 列出任务 |

**启用条件**: `isTodoV2Enabled()`

## 网络工具

### WebFetchTool

**文件**: [src/tools/WebFetchTool/WebFetchTool.ts](src/tools/WebFetchTool/WebFetchTool.ts)

| 属性 | 值 |
|------|------|
| name | `WebFetch` |
| 只读 | ✓ true |
| 并发安全 | ✓ true |

**输入 Schema**:
```typescript
z.object({
  url: z.string().describe("URL"),
  prompt: z.string().describe("提取提示")
})
```

**特殊功能**:
- HTML → Markdown 转换
- 支持重定向跟随
- 15 分钟缓存
- 认证 URL 限制

### WebSearchTool

**文件**: [src/tools/WebSearchTool/WebSearchTool.ts](src/tools/WebSearchTool/WebSearchTool.ts)

| 属性 | 值 |
|------|------|
| name | `WebSearch` |
| 只读 | ✓ true |
| 并发安全 | ✓ true |

**输入 Schema**:
```typescript
z.object({
  query: z.string().describe("搜索查询"),
  allowed_domains: z.array(z.string()).optional(),
  blocked_domains: z.array(z.string()).optional()
})
```

## 交互工具

### AskUserQuestionTool

**文件**: [src/tools/AskUserQuestionTool/AskUserQuestionTool.ts](src/tools/AskUserQuestionTool/AskUserQuestionTool.ts)

| 属性 | 值 |
|------|------|
| name | `AskUserQuestion` |
| 只读 | ✓ true |

**输入 Schema**:
```typescript
z.object({
  questions: z.array(z.object({
    question: z.string().describe("问题"),
    header: z.string().describe("简短标签"),
    options: z.array(z.object({
      label: z.string(),
      description: z.string(),
      preview: z.string().optional()
    })),
    multiSelect: z.boolean()
  }))
})
```

**特殊功能**:
- 支持预览展示
- 多选支持
- 自定义输入选项

## 规划工具

### EnterPlanModeTool

**文件**: [src/tools/EnterPlanModeTool/EnterPlanModeTool.ts](src/tools/EnterPlanModeTool/EnterPlanModeTool.ts)

| 属性 | 值 |
|------|------|
| name | `EnterPlanMode` |

### ExitPlanModeV2Tool

**文件**: [src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts](src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts)

| 属性 | 值 |
|------|------|
| name | `ExitPlanMode` |

**输入 Schema**:
```typescript
z.object({
  allowedPrompts: z.array(z.object({
    tool: z.enum(['Bash']),
    prompt: z.string()
  })).optional()
})
```

## 工作树工具

### EnterWorktreeTool

**文件**: [src/tools/EnterWorktreeTool/EnterWorktreeTool.ts](src/tools/EnterWorktreeTool/EnterWorktreeTool.ts)

| 属性 | 值 |
|------|------|
| name | `EnterWorktree` |
| 启用条件 | `isWorktreeModeEnabled()` |

### ExitWorktreeTool

**文件**: [src/tools/ExitWorktreeTool/ExitWorktreeTool.ts](src/tools/ExitWorktreeTool/ExitWorktreeTool.ts)

| 属性 | 值 |
|------|------|
| name | `ExitWorktree` |
| 启用条件 | `isWorktreeModeEnabled()` |

## 其他工具

### SkillTool

**文件**: [src/tools/SkillTool/SkillTool.ts](src/tools/SkillTool/SkillTool.ts)

| 属性 | 值 |
|------|------|
| name | `Skill` |

**输入 Schema**:
```typescript
z.object({
  skill: z.string().describe("技能名称"),
  args: z.string().optional()
})
```

### NotebookEditTool

**文件**: [src/tools/NotebookEditTool/NotebookEditTool.ts](src/tools/NotebookEditTool/NotebookEditTool.ts)

| 属性 | 值 |
|------|------|
| name | `NotebookEdit` |

**输入 Schema**:
```typescript
z.object({
  notebook_path: z.string().describe("绝对路径"),
  cell_id: z.string().optional(),
  cell_type: z.enum(['code', 'markdown']).optional(),
  edit_mode: z.enum(['replace', 'insert', 'delete']).optional(),
  new_source: z.string().describe("新内容")
})
```

### LSPTool

**文件**: [src/tools/LSPTool/LSPTool.ts](src/tools/LSPTool/LSPTool.ts)

| 属性 | 值 |
|------|------|
| name | `LSP` |
| 启用条件 | `ENABLE_LSP_TOOL=true` |

**输入 Schema**:
```typescript
z.object({
  operation: z.enum(['definition', 'references', 'hover', 'rename']),
  path: z.string(),
  line: z.number(),
  character: z.number()
})
```

### ConfigTool

**文件**: [src/tools/ConfigTool/ConfigTool.ts](src/tools/ConfigTool/ConfigTool.ts)

| 属性 | 值 |
|------|------|
| name | `Config` |
| 启用条件 | `USER_TYPE === 'ant'` |

### MCP 工具

| 工具 | 说明 |
|------|------|
| ListMcpResourcesTool | 列出 MCP 资源 |
| ReadMcpResourceTool | 读取 MCP 资源 |
| MCPTool | 调用 MCP 工具 |

### 其他条件工具

| 工具 | 启用条件 |
|------|----------|
| TungstenTool | `USER_TYPE === 'ant'` |
| WebBrowserTool | `WEB_BROWSER_TOOL` feature |
| SnipTool | `HISTORY_SNIP` feature |
| WorkflowTool | `WORKFLOW_SCRIPTS` feature |
| SleepTool | `PROACTIVE` 或 `KAIROS` feature |
| CronCreate/Delete/ListTool | `AGENT_TRIGGERS` feature |
| MonitorTool | `MONITOR_TOOL` feature |
| SendMessageTool | Agent Swarms 启用时 |
| TeamCreateTool | Agent Swarms 启用时 |
| TeamDeleteTool | Agent Swarms 启用时 |

## 工具属性汇总

### 并发安全性

| 安全 | 不安全 |
|------|--------|
| Read, Glob, Grep | Write, Edit（同文件） |
| WebFetch, WebSearch | Bash（部分） |
| Agent, TaskOutput | PowerShell |

### 只读性

| 只读 | 写入 |
|------|------|
| Read, Glob, Grep | Write, Edit |
| WebFetch, WebSearch | Bash（取决于命令） |
| TaskOutput | NotebookEdit |

### 破坏性

| 破坏性 | 非破坏性 |
|--------|----------|
| Write | Read, Edit, Glob, Grep |
| Bash（部分命令） | WebFetch, WebSearch |

## 工具过滤

### filterToolsByDenyRules

```typescript
export function filterToolsByDenyRules<T>(
  tools: readonly T[],
  permissionContext: ToolPermissionContext
): T[] {
  return tools.filter(tool => !getDenyRuleForTool(permissionContext, tool))
}
```

### assembleToolPool

```typescript
export function assembleToolPool(
  permissionContext: ToolPermissionContext,
  mcpTools: Tools
): Tools {
  const builtInTools = getTools(permissionContext)
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)
  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
    'name'
  )
}
```

## 关键文件

- [src/tools.ts](src/tools.ts) — 工具注册中心
- [src/Tool.ts](src/Tool.ts) — Tool 接口定义
- [src/tools/](src/tools/) — 各工具实现目录
- [src/constants/tools.js](src/constants/tools.js) — 工具常量

## 关联文档

- [05-工具系统](05-tool-system.md)
- [07-权限系统](07-permission-system.md)