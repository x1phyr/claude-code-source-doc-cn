# 常量参考

## 概述

本文档汇总 Claude Code 的核心常量定义。

## 工具名称常量

### 定义位置

[src/tools/BashTool/toolName.ts](src/tools/BashTool/toolName.ts)

```typescript
export const BASH_TOOL_NAME = 'Bash'
export const FILE_READ_TOOL_NAME = 'Read'
export const FILE_EDIT_TOOL_NAME = 'Edit'
export const FILE_WRITE_TOOL_NAME = 'Write'
export const GLOB_TOOL_NAME = 'Glob'
export const GREP_TOOL_NAME = 'Grep'
export const AGENT_TOOL_NAME = 'Agent'
export const TASK_OUTPUT_TOOL_NAME = 'TaskOutput'
export const WEB_FETCH_TOOL_NAME = 'WebFetch'
export const WEB_SEARCH_TOOL_NAME = 'WebSearch'
export const TODO_WRITE_TOOL_NAME = 'TodoWrite'
export const ASK_USER_QUESTION_TOOL_NAME = 'AskUserQuestion'
export const SKILL_TOOL_NAME = 'Skill'
export const ENTER_PLAN_MODE_TOOL_NAME = 'EnterPlanMode'
export const EXIT_PLAN_MODE_TOOL_NAME = 'ExitPlanMode'
```

## XML 标签常量

### 定义位置

[src/constants/xmlTags.ts](src/constants/xmlTags.ts)

```typescript
export const XML_TAGS = {
  THINKING: 'thinking',
  IGNORE: 'ignore',
  HIGHLIGHT: 'highlight',
  SUGGESTION: 'suggestion',
  EXAMPLE: 'example',
  // ...
}
```

## API 限制常量

### 定义位置

[src/constants/apiLimits.ts](src/constants/apiLimits.ts)

```typescript
export const MAX_OUTPUT_TOKENS = 8192
export const MAX_CONTEXT_TOKENS = 200000
export const PDF_MAX_PAGES_PER_READ = 20
export const PDF_EXTRACT_SIZE_THRESHOLD = 10 * 1024 * 1024
export const IMAGE_MAX_SIZE = 20 * 1024 * 1024
```

## 工具限制常量

### 定义位置

[src/constants/toolLimits.ts](src/constants/toolLimits.ts)

```typescript
export const TOOL_SUMMARY_MAX_LENGTH = 200
export const BASH_DEFAULT_TIMEOUT_MS = 120000
export const BASH_MAX_TIMEOUT_MS = 600000
export const GLOB_MAX_RESULTS = 1000
export const GREP_MAX_RESULTS = 250
```

## 文件扩展名常量

### 定义位置

[src/constants/files.ts](src/constants/files.ts)

```typescript
export const BINARY_EXTENSIONS = [
  '.png', '.jpg', '.jpeg', '.gif', '.bmp', '.ico',
  '.pdf', '.zip', '.tar', '.gz',
  '.exe', '.dll', '.so', '.dylib',
  // ...
]

export const IMAGE_EXTENSIONS = [
  '.png', '.jpg', '.jpeg', '.gif', '.bmp', '.webp', '.ico'
]

export const PDF_EXTENSIONS = ['.pdf']
```

## 配置文件路径常量

### 定义位置

[src/utils/settings/constants.ts](src/utils/settings/constants.ts)

```typescript
export const USER_SETTINGS_PATH = '~/.claude/settings.json'
export const PROJECT_SETTINGS_PATH = '.claude/settings.json'
export const LOCAL_SETTINGS_PATH = '.claude/settings.local.json'
export const MEMORY_DIR = '.claude/memory/'
export const SESSIONS_DIR = '~/.claude/sessions/'
```

## 查询来源常量

### 定义位置

[src/utils/promptCategory.ts](src/utils/promptCategory.ts)

```typescript
export const QUERY_SOURCES = {
  USER: 'user',
  SUGGESTION: 'suggestion',
  COMMAND: 'command',
  RESUME: 'resume',
  INIT: 'init',
  // ...
}
```

## 权限模式常量

### 定义位置

[src/utils/permissions/PermissionMode.ts](src/utils/permissions/PermissionMode.ts)

```typescript
export const PERMISSION_MODES = {
  DEFAULT: 'default',
  ACCEPT_EDITS: 'acceptEdits',
  BYPASS_PERMISSIONS: 'bypassPermissions',
  DONT_ASK: 'dontAsk',
  PLAN: 'plan',
  AUTO: 'auto'
} as const
```

## 模型常量

### 定义位置

[src/utils/model/canonicalNames.ts](src/utils/model/canonicalNames.ts)

```typescript
export const MODEL_ALIASES = {
  'claude-opus-4-6': 'opus',
  'claude-sonnet-4-6': 'sonnet',
  'claude-haiku-4-5': 'haiku',
  // ...
}

export const MODEL_IDS = {
  OPUS: 'claude-opus-4-6-20250801',
  SONNET: 'claude-sonnet-4-6-20250801',
  HAIKU: 'claude-haiku-4-5-20251001'
}
```

## 颜色常量

### 定义位置

[src/tools/AgentTool/agentColorManager.ts](src/tools/AgentTool/agentColorManager.ts)

```typescript
export const AGENT_COLORS = [
  'cyan',
  'magenta',
  'yellow',
  'blue',
  'green',
  'red'
] as const

export type AgentColorName = typeof AGENT_COLORS[number]
```

## 退出代码常量

```typescript
export const EXIT_CODES = {
  SUCCESS: 0,
  ERROR: 1,
  INTERRUPT: 130,
  TIMEOUT: 124
}
```

## 环境变量常量

```typescript
export const ENV_VARS = {
  ANTHROPIC_API_KEY: 'ANTHROPIC_API_KEY',
  CLAUDE_CODE_MODEL: 'CLAUDE_CODE_MODEL',
  CLAUDE_CODE_SIMPLE: 'CLAUDE_CODE_SIMPLE',
  CLAUDE_CODE_DISABLE_BACKGROUND_TASKS: 'CLAUDE_CODE_DISABLE_BACKGROUND_TASKS',
  // ...
}
```

## 沙箱常量

### 定义位置

[src/tools/BashTool/shouldUseSandbox.ts](src/tools/BashTool/shouldUseSandbox.ts)

```typescript
export const SANDBOX_BLOCKED_COMMANDS = [
  'rm -rf /',
  'mkfs',
  'dd if=',
  // ...
]

export const SANDBOX_PROTECTED_PATHS = [
  '/',
  '/etc',
  '/usr',
  '/bin',
  // ...
]
```

## 消息类型常量

```typescript
export const MESSAGE_TYPES = {
  USER: 'user',
  ASSISTANT: 'assistant',
  SYSTEM: 'system',
  PROGRESS: 'progress',
  TOOL_USE: 'tool_use',
  TOOL_RESULT: 'tool_result'
}
```

## 关键文件

- [src/constants/](src/constants/) — 常量目录
- [src/tools/BashTool/toolName.ts](src/tools/BashTool/toolName.ts) — 工具名
- [src/constants/apiLimits.ts](src/constants/apiLimits.ts) — API 限制
- [src/constants/files.ts](src/constants/files.ts) — 文件常量

## 关联文档

- [21-类型参考](21-types-reference.md)
- [23-配置参考](23-configuration-reference.md)