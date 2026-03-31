# 目录结构详解

## 概述

本文档详细说明 Claude Code 源码的目录结构和各模块职责。

## 根目录结构

```
claude-code-rev/
├── src/                    # 核心源代码
├── shims/                  # 本地 shim 包（替代不可用的原生模块）
├── vendor/                 # 第三方代码
├── docs/                   # 文档（本系列）
├── package.json            # 项目配置
├── tsconfig.json           # TypeScript 配置
├── bun.lock               # Bun 锁文件
└── README.md              # 项目说明
```

## src/ 目录结构

### 核心入口文件

| 文件 | 说明 |
|------|------|
| `bootstrap-entry.ts` | 最顶层入口，导入并启动 CLI |
| `bootstrapMacro.ts` | 注入构建时常量（VERSION 等） |
| `main.tsx` | REPL 主渲染器（800KB+，核心 UI） |
| `QueryEngine.ts` | 查询引擎类，管理会话状态 |
| `query.ts` | 查询主循环（1700+ 行） |
| `tools.ts` | 工具注册和聚合 |
| `commands.ts` | 命令注册中心 |
| `context.ts` | 系统和用户上下文构建 |
| `Tool.ts` | Tool 类型定义和工厂函数 |
| `Task.ts` | 任务类型定义 |
| `tasks.ts` | 任务管理 |
| `query.ts` | 查询处理 |

### 子目录说明

#### `src/entrypoints/` - 入口点

| 文件 | 说明 |
|------|------|
| `cli.tsx` | CLI 主入口，参数解析和模式分发 |
| `init.ts` | 初始化逻辑 |
| `mcp.ts` | MCP 入口 |
| `agentSdkTypes.ts` | Agent SDK 类型定义 |
| `sandboxTypes.ts` | 沙箱类型定义 |
| `sdk/` | SDK 相关类型定义 |

#### `src/tools/` - 工具实现（87 个子目录）

工具按功能分组，每个工具有独立目录，包含：
- 主文件（如 `BashTool.ts`）
- `prompt.ts` - 工具描述和提示
- 辅助文件（如验证、权限检查）

主要工具：

| 工具目录 | 说明 |
|----------|------|
| `BashTool/` | Shell 命令执行 |
| `FileReadTool/` | 文件读取 |
| `FileEditTool/` | 文件编辑 |
| `FileWriteTool/` | 文件写入 |
| `GlobTool/` | 文件模式匹配 |
| `GrepTool/` | 内容搜索 |
| `AgentTool/` | 子代理管理 |
| `TaskCreateTool/` 等 | 任务管理工具集 |
| `WebFetchTool/` | 网页获取 |
| `WebSearchTool/` | 网页搜索 |
| `AskUserQuestionTool/` | 用户交互 |
| `EnterPlanModeTool/` | 进入规划模式 |
| `ExitPlanModeTool/` | 退出规划模式 |
| `EnterWorktreeTool/` | 进入工作树 |
| `ExitWorktreeTool/` | 退出工作树 |
| `TodoWriteTool/` | 待办事项管理 |
| `LSPTool/` | LSP 集成 |
| `MCPTool/` | MCP 工具调用 |
| `ConfigTool/` | 配置管理 |
| `NotebookEditTool/` | Jupyter Notebook 编辑 |
| `SkillTool/` | 技能调用 |
| `PowerShellTool/` | PowerShell 命令执行 |
| `SleepTool/` | 延迟工具 |
| `SendUserFileTool/` | 发送文件给用户 |
| `PushNotificationTool/` | 推送通知 |

#### `src/commands/` - 命令实现（54 个子目录）

每个命令有独立目录，导出 `Command` 对象。

主要命令：

| 命令目录 | 说明 |
|----------|------|
| `add-dir/` | 添加工作目录 |
| `agents/` | 代理管理 |
| `branch/` | Git 分支操作 |
| `bridge/` | 远程桥接 |
| `chrome/` | Chrome 集成 |
| `clear/` | 清屏 |
| `commit/` | Git 提交 |
| `compact/` | 上下文压缩 |
| `config/` | 配置管理 |
| `context/` | 上下文管理 |
| `cost/` | 成本统计 |
| `diff/` | 差异查看 |
| `doctor/` | 诊断工具 |
| `export/` | 导出会话 |
| `fast/` | 快速模式 |
| `feedback/` | 反馈 |
| `files/` | 文件列表 |
| `help/` | 帮助信息 |
| `hooks/` | 钩子管理 |
| `init/` | 初始化 |
| `keybindings/` | 快捷键管理 |
| `login/`, `logout/` | 认证 |
| `mcp/` | MCP 管理 |
| `memory/` | 记忆系统 |
| `model/` | 模型选择 |
| `permissions/` | 权限管理 |
| `plan/` | 规划模式 |
| `resume/` | 恢复会话 |
| `session/` | 会话管理 |
| `skills/` | 技能管理 |
| `status/` | 状态显示 |
| `theme/` | 主题管理 |
| `upgrade/` | 升级 |
| `vim/` | Vim 模式 |

#### `src/services/` - 服务层

| 子目录 | 说明 |
|--------|------|
| `api/` | API 客户端、请求构建、错误处理 |
| `mcp/` | MCP 连接管理、工具注册、OAuth |
| `compact/` | 上下文压缩策略（AutoCompact、MicroCompact） |
| `lsp/` | LSP 客户端集成 |
| `analytics/` | 分析和追踪（GrowthBook、DataDog） |
| `SessionMemory/` | 会话记忆 |
| `PromptSuggestion/` | 提示建议 |
| `MagicDocs/` | 文档魔法 |
| `autoDream/` | 自动思考 |
| `toolUseSummary/` | 工具调用摘要 |
| `skillSearch/` | 技能搜索 |

#### `src/components/` - UI 组件（100+ 文件）

使用 Ink 框架构建的终端 UI 组件。

| 子目录/文件 | 说明 |
|-------------|------|
| `App.tsx` | 应用根组件 |
| `Message.tsx` | 消息渲染 |
| `MessageRow.tsx` | 消息行 |
| `HelpV2/` | 帮助界面 |
| `LogoV2/` | Logo 和欢迎界面 |
| `Markdown.tsx` | Markdown 渲染 |
| `HighlightedCode.tsx` | 代码高亮 |
| `FeedbackSurvey/` | 反馈调查 |
| `CustomSelect/` | 自定义选择器 |
| `AutoUpdater.tsx` | 自动更新 |
| `Dialog` 系列 | 各种对话框 |

#### `src/hooks/` - React Hooks（80+ 文件）

| 文件 | 说明 |
|------|------|
| `useSettings.ts` | 设置管理 |
| `useMergedTools.ts` | 工具合并 |
| `useMergedCommands.ts` | 命令合并 |
| `useCanUseTool.tsx` | 权限检查 |
| `useReplBridge.tsx` | 远程会话 |
| `useMainLoopModel.ts` | 模型选择 |
| `useTextInput.ts` | 文本输入 |
| `useGlobalKeybindings.tsx` | 全局快捷键 |
| `useCommandKeybindings.tsx` | 命令快捷键 |
| `notifs/` | 通知相关 hooks |
| `toolPermission/` | 工具权限 hooks |

#### `src/utils/` - 工具函数（100+ 文件）

| 子目录/文件 | 说明 |
|-------------|------|
| `bash/` | Shell 解析与执行（AST、解析器、引用） |
| `permissions/` | 权限处理 |
| `model/` | 模型配置 |
| `git/` | Git 操作 |
| `claudeInChrome/` | Chrome 集成 |
| `computerUse/` | Computer Use 功能 |
| `hooks/` | Hook 执行 |
| `plugins/` | 插件加载 |
| `bash/` | Bash 解析 |
| `api.ts` | API 工具 |
| `auth.ts` | 认证工具 |
| `config.ts` | 配置工具 |
| `cwd.ts` | 工作目录 |
| `git.ts` | Git 工具 |
| `log.ts` | 日志工具 |
| `messages.ts` | 消息工具 |

#### `src/types/` - 类型定义

| 文件 | 说明 |
|------|------|
| `message.ts` | 消息类型家族 |
| `permissions.ts` | 权限类型 |
| `command.ts` | 命令类型 |
| `tools.ts` | 工具类型 |
| `hooks.ts` | Hook 类型 |
| `ids.ts` | ID 类型 |
| `logs.ts` | 日志类型 |
| `plugin.ts` | 插件类型 |
| `generated/` | 自动生成的类型 |

#### `src/skills/` - 技能系统

| 子目录/文件 | 说明 |
|-------------|------|
| `bundledSkills.ts` | 内置技能注册 |
| `loadSkillsDir.ts` | 技能目录加载 |
| `mcpSkills.ts` | MCP 技能 |
| `bundled/` | 内置技能内容 |
| `bundled/claude-api/` | Claude API 文档技能 |
| `bundled/verify/` | 验证技能 |

#### `src/bridge/` - 远程桥接

用于远程会话和移动端连接。

| 文件 | 说明 |
|------|------|
| `bridgeMain.ts` | 桥接主逻辑 |
| `bridgeApi.ts` | API 定义 |
| `bridgeConfig.ts` | 配置 |
| `bridgeMessaging.ts` | 消息处理 |
| `replBridge.ts` | REPL 桥接 |
| `remoteBridgeCore.ts` | 远程核心 |

#### `src/cli/` - CLI 处理器

| 文件 | 说明 |
|------|------|
| `bg.ts` | 后台会话管理 |
| `handlers/` | 各种命令处理器 |
| `exit.ts` | 退出处理 |

#### `src/state/` - 状态管理

| 文件 | 说明 |
|------|------|
| `AppState.tsx` | 全局状态结构 |
| `AppContext.tsx` | React Context |

#### `src/constants/` - 常量定义

| 文件 | 说明 |
|------|------|
| `prompts.ts` | 系统提示模板 |
| `common.ts` | 通用常量 |
| `xml.ts` | XML 标签常量 |
| `querySource.ts` | 查询来源 |

#### `src/query/` - 查询相关

| 文件 | 说明 |
|------|------|
| `config.ts` | 查询配置 |
| `deps.ts` | 依赖注入 |
| `transitions.ts` | 状态转换 |
| `stopHooks.ts` | 停止钩子 |
| `tokenBudget.ts` | Token 预算 |

#### 其他目录

| 目录 | 说明 |
|------|------|
| `assistant/` | Assistant 功能 |
| `buddy/` | Buddy 功能（动画精灵） |
| `context/` | 上下文管理 |
| `coordinator/` | 协调器模式 |
| `daemon/` | 守护进程 |
| `jobs/` | 任务系统 |
| `keybindings/` | 快捷键定义 |
| `migrations/` | 数据迁移 |
| `native-ts/` | 原生模块 TypeScript 实现 |
| `outputStyles/` | 输出样式 |
| `plugins/` | 插件系统 |
| `proactive/` | 主动功能 |
| `remote/` | 远程功能 |
| `screens/` | 屏幕组件 |
| `ssh/` | SSH 支持 |
| `vim/` | Vim 模式 |
| `voice/` | 语音模式 |

## shims/ 目录

本地 shim 包，替代不可用的原生模块：

| 目录 | 说明 |
|------|------|
| `ant-claude-for-chrome-mcp/` | Chrome MCP shim |
| `ant-computer-use-input/` | Computer Use 输入 shim |
| `ant-computer-use-mcp/` | Computer Use MCP shim |
| `ant-computer-use-swift/` | Computer Use Swift shim |
| `color-diff-napi/` | 颜色差异原生模块 shim |
| `modifiers-napi/` | 修饰符原生模块 shim |
| `url-handler-napi/` | URL 处理原生模块 shim |

## 关联文档

- [01-架构总览](01-architecture-overview.md)
- [03-启动流程](03-entry-and-bootstrap.md)