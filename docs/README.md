# Claude Code 学习文档导航

## 概述

本文档库为 Claude Code 还原项目提供完整的学习指南，涵盖架构、工具、命令、权限、MCP、API 等所有核心模块。

## 目录结构

```
docs/
├── 01-入门/           基础入门（架构、目录、启动、术语）
├── 02-核心引擎/       查询引擎、状态管理、消息处理
├── 03-工具系统/       工具框架、工具清单、权限检查、沙箱
├── 04-命令系统/       命令框架、命令清单、输入处理
├── 05-权限安全/       权限系统、权限检查、沙箱、危险检测
├── 06-MCP协议/        MCP集成、协议实现、OAuth、服务器开发
├── 07-Bridge与远程/   Bridge远程控制、远程会话管理
├── 08-上下文管理/     上下文压缩、Token估算、缓存策略
├── 09-API与网络/      API服务、重试机制、错误处理
├── 10-Skills系统/     Skills系统、技能开发指南
├── 11-插件系统/       插件系统、Hook系统
├── 12-代理系统/       多代理协调、预测执行
├── 13-Memory系统/     Memory系统、CLAUDE.md系统
├── 14-服务层/         Compact服务、LSP服务、分析服务
├── 15-UI组件/         核心组件、终端渲染、动画、对话框
├── 16-响应式系统/     Ink响应式框架详解
├── 17-Hooks清单/      React Hooks清单、工具函数清单
├── 18-调试测试/       调试工具、测试策略
└── 19-参考手册/       类型、常量、配置参考
```

## 文档列表

### 01-入门

| 文档 | 说明 |
|------|------|
| [架构总览](01-入门/架构总览-Architecture-Overview.md) | 项目整体架构和核心模块 |
| [目录结构](01-入门/目录结构-Directory-Structure.md) | 源码目录组织和文件分布 |
| [启动流程](01-入门/启动流程-Entry-And-Bootstrap.md) | CLI启动和Bootstrap过程 |
| [术语表](01-入门/术语表-Glossary.md) | 专用术语和缩写定义 |

### 02-核心引擎

| 文档 | 说明 |
|------|------|
| [查询引擎](02-核心引擎/查询引擎-Query-Engine.md) | QueryEngine核心调度逻辑 |
| [状态管理](02-核心引擎/状态管理-State-Management.md) | AppState和Store状态管理 |
| [消息处理流程](02-核心引擎/消息处理流程-Message-Processing-Flow.md) | 消息类型和处理流程 ⏳ |

### 03-工具系统

| 文档 | 说明 |
|------|------|
| [工具系统详解](03-工具系统/工具系统详解-Tool-System.md) | 工具框架和生命周期 |
| [工具清单](03-工具系统/工具清单-Tools-Catalog.md) | 所有工具详细说明 |
| [权限检查细节](03-工具系统/权限检查细节-Permission-Check-Details.md) | 权限检查流程 ⏳ |
| [沙箱实现](03-工具系统/沙箱实现-Sandbox-Implementation.md) | 沙箱隔离机制 ⏳ |

### 04-命令系统

| 文档 | 说明 |
|------|------|
| [命令系统详解](04-命令系统/命令系统详解-Command-System.md) | 命令框架和类型 |
| [命令清单](04-命令系统/命令清单-Commands-Catalog.md) | 所有斜杠命令说明 |
| [输入处理](04-命令系统/输入处理-Input-Handling.md) | 输入解析和处理 ⏳ |

### 05-权限安全

| 文档 | 说明 |
|------|------|
| [权限系统详解](05-权限安全/权限系统详解-Permission-System.md) | 权限模式和规则 |
| [权限检查细节](05-权限安全/权限检查细节-Permission-Check-Details.md) | → 链接至03-工具系统 |
| [沙箱实现](05-权限安全/沙箱实现-Sandbox-Implementation.md) | → 链接至03-工具系统 |
| [危险命令检测](05-权限安全/危险命令检测-Dangerous-Command-Detection.md) | 危险命令识别 ⏳ |

### 06-MCP协议

| 文档 | 说明 |
|------|------|
| [MCP集成详解](06-MCP协议/MCP集成详解-MCP-Integration.md) | MCP服务器集成 |
| [MCP协议实现](06-MCP协议/MCP协议实现-MCP-Protocol-Implementation.md) | 协议细节 ⏳ |
| [OAuth认证流程](06-MCP协议/OAuth认证流程-OAuth-Authentication-Flow.md) | OAuth认证 ⏳ |
| [MCP服务器开发](06-MCP协议/MCP服务器开发-MCP-Server-Development.md) | 开发指南 ⏳ |

### 07-Bridge与远程

| 文档 | 说明 |
|------|------|
| [Bridge远程控制](07-Bridge与远程/Bridge远程控制-Bridge-Remote-Control.md) | Bridge模式 ⏳ |
| [远程会话管理](07-Bridge与远程/远程会话管理-Remote-Session-Management.md) | 远程会话 ⏳ |

### 08-上下文管理

| 文档 | 说明 |
|------|------|
| [上下文压缩算法](08-上下文管理/上下文压缩算法-Context-Compression-Algorithms.md) | 压缩策略 ⏳ |
| [Token估算机制](08-上下文管理/Token估算机制-Token-Estimation.md) | Token计算 ⏳ |
| [缓存策略](08-上下文管理/缓存策略-Caching-Strategies.md) | 缓存机制 ⏳ |

### 09-API与网络

| 文档 | 说明 |
|------|------|
| [API服务详解](09-API与网络/API服务详解-API-Service.md) | API客户端 ⏳ |
| [API重试机制](09-API与网络/API重试机制-API-Retry-Mechanism.md) | 重试策略 ⏳ |
| [错误处理模式](09-API与网络/错误处理模式-Error-Handling-Patterns.md) | 错误处理 ⏳ |

### 10-Skills系统

| 文档 | 说明 |
|------|------|
| [Skills系统详解](10-Skills系统/Skills系统详解-Skills-System.md) | Skills框架 |
| [技能开发指南](10-Skills系统/技能开发指南-Skill-Development-Guide.md) | 开发指南 ⏳ |

### 11-插件系统

| 文档 | 说明 |
|------|------|
| [插件系统](11-插件系统/插件系统-Plugin-System.md) | 插件框架 ⏳ |
| [Hook系统详解](11-插件系统/Hook系统详解-Hooks-System.md) | CLI Hooks |

### 12-代理系统

| 文档 | 说明 |
|------|------|
| [多代理协调](12-代理系统/多代理协调-Multi-Agent-Coordination.md) | Agent协作 ⏳ |
| [预测执行](12-代理系统/预测执行-Speculation-Prediction.md) | Speculation机制 ⏳ |

### 13-Memory系统

| 文档 | 说明 |
|------|------|
| [Memory系统详解](13-Memory系统/Memory系统详解-Memory-System.md) | 记忆持久化 |
| [CLAUDE.md系统详解](13-Memory系统/CLAUDE.md系统详解-CLAUDE-MD-System.md) | 项目指令文件 |

### 14-服务层

| 文档 | 说明 |
|------|------|
| [服务层详解](14-服务层/服务层详解-Services-Detail.md) | 服务总览 |
| [Compact服务](14-服务层/Compact服务-Compact-Service.md) | Compact逻辑 ⏳ |
| [LSP服务](14-服务层/LSP服务-LSP-Service.md) | LSP集成 ⏳ |
| [分析服务](14-服务层/分析服务-Analytics-Service.md) | 统计分析 ⏳ |

### 15-UI组件

| 文档 | 说明 |
|------|------|
| [UI组件详解](15-UI组件/UI组件详解-Components-Detail.md) | 组件总览 |
| [核心组件](15-UI组件/核心组件-Core-Components.md) | 核心UI组件 ⏳ |
| [终端渲染](15-UI组件/终端渲染-Terminal-Rendering.md) | Ink渲染 ⏳ |
| [动画效果](15-UI组件/动画效果-Animation-Effects.md) | 动画系统 ⏳ |
| [对话框组件](15-UI组件/对话框组件-Dialog-Components.md) | 对话框 ⏳ |

### 16-响应式系统

| 文档 | 说明 |
|------|------|
| [响应式系统详解](16-响应式系统/响应式系统详解-Reactive-System.md) | Ink框架响应式 |

### 17-Hooks清单

| 文档 | 说明 |
|------|------|
| [Hooks清单](17-Hooks清单/Hooks清单-Hooks-Catalog.md) | React Hooks列表 |
| [工具函数清单](17-Hooks清单/工具函数清单-Utils-Catalog.md) | 工具函数列表 |

### 18-调试测试

| 文档 | 说明 |
|------|------|
| [调试工具](18-调试测试/调试工具-Debugging-Tools.md) | 调试方法 ⏳ |
| [测试策略](18-调试测试/测试策略-Testing-Strategy.md) | 测试方案 ⏳ |

### 19-参考手册

| 文档 | 说明 |
|------|------|
| [类型参考](19-参考手册/类型参考-Types-Reference.md) | 核心类型定义 |
| [常量参考](19-参考手册/常量参考-Constants-Reference.md) | 常量汇总 |
| [配置参考](19-参考手册/配置参考-Configuration-Reference.md) | 配置选项 |

---

⏳ = 待编写文档

## 快速导航

**新手入门**：建议从 01-入门 目录开始，按顺序阅读架构总览、目录结构、启动流程。

**开发者参考**：重点关注 03-工具系统、04-命令系统、19-参考手册。

**深度学习**：02-核心引擎、06-MCP协议、08-上下文管理 提供深入的底层实现细节。