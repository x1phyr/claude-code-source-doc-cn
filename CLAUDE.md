# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目背景

这是一个**还原后的 Claude Code 源码树**，主要通过 source map 逆向还原并补齐缺失模块。它并非上游仓库的原始状态。部分文件无法恢复，已用兼容 shim 或降级实现替代。

## 构建与开发命令

使用 Bun 进行本地开发。环境要求：Bun 1.3.5+，Node.js 24+。

```bash
bun install          # 安装依赖和本地 shim 包
bun run dev          # 启动还原后的 CLI 入口（交互模式）
bun run start        # dev 的别名
bun run version      # 验证 CLI 启动并打印版本号
```

仓库目前没有正式的 `lint` 或 `test` 脚本。修改后手动验证：启动 CLI 并测试相关路径。

## 架构概览

### 入口点
- `src/bootstrap-entry.ts` → `src/entrypoints/cli.tsx` — 主 CLI 启动链
- `src/bootstrapMacro.ts` — 注入构建时常量（VERSION、BUILD_TIME 等）
- `src/main.tsx` — 基于 Ink 的 TUI 渲染器（800KB+）
- `src/commands.ts` — 命令注册中心，聚合所有斜杠命令

### 核心系统
- **Tools** (`src/tools/`) — 模块化工具实现（BashTool、FileEditTool、AgentTool、MCPTool 等）。每个工具有独立子目录，含 prompt、校验和辅助函数。
- **Commands** (`src/commands/`) — 斜杠命令实现，按功能组织（如 `mcp/`、`config/`、`memory/`）。导出 `Command` 对象的 TypeScript 模块。
- **Services** (`src/services/`) — 后端服务：API 客户端 (`api/`)、MCP 管理 (`mcp/`)、LSP 集成 (`lsp/`)、分析统计、会话记忆、compact 逻辑。
- **Components** (`src/components/`) — 基于 Ink 的 React 组件，用于 TUI 渲染。
- **Skills** (`src/skills/`) — 内置 skills 和 skill 加载基础设施。

### Shim 包
`shims/` 目录包含本地包，从 `src/native-ts/` 重导出或提供降级 fallback：
- `ant-computer-use-mcp` — Computer Use MCP 的会话审批 shim（原生桌面操作不可用）
- `color-diff-napi`、`modifiers-napi`、`url-handler-napi` — 原生模块重导出

### 命令类型
`src/commands.ts` 中命令分为：
- `prompt` — 展开为发送给模型的文本（skills、workflows）
- `local` — 本地执行，返回文本输出
- `local-jsx` — 渲染 Ink UI 组件

## 编码风格

TypeScript 优先，ESM 导入，`react-jsx`。保持与周围文件风格一致：
- 多数文件省略分号，使用单引号
- 变量/函数用描述性 camelCase
- React 组件和管理类用 PascalCase
- 有注释警告禁止重排序时保持导入顺序稳定（ANT-ONLY 标记）
- 倾向小而专注的模块，避免大杂烩式工具文件

## 还原约束

这是重建源码，不是原始上游代码。修改时：
- 倾向最小、可审计的改动
- 对因恢复 fallback 而添加的 workaround 做文档记录
- `shims/` 中的原生绑定可能有降级行为 — 使用前检查 shim 注释
- 部分 feature-flag 控制的命令（`feature('KAIROS')`、`feature('BRIDGE_MODE')`）可能无法完全工作

## 关键文件

- `src/commands.ts` — 命令注册中心，导入所有命令
- `src/tools.ts` — REPL 工具聚合
- `src/QueryEngine.ts` — 查询执行调度
- `src/Task.ts`、`src/tasks.ts` — 任务管理
- `src/context.ts` — 共享应用状态上下文
- `src/skills/bundledSkills.ts` — 内置 skill 注册