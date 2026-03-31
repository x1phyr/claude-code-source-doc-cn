# 状态管理详解

## 概述

Claude Code 使用 React Context + 自定义 Store 模式管理全局状态。核心是 `AppState` 类型和 `AppStateStore`，配合 `useSyncExternalStore` 实现高效的订阅式状态更新。

## Store 模式

### 定义位置

[src/state/store.ts](src/state/store.ts)

### 核心结构

```typescript
type Listener = () => void
type OnChange<T> = (args: { newState: T; oldState: T }) => void

export type Store<T> = {
  getState: () => T                              // 获取当前状态
  setState: (updater: (prev: T) => T) => void    // 更新状态
  subscribe: (listener: Listener) => () => void  // 订阅变化
}

export function createStore<T>(
  initialState: T,
  onChange?: OnChange<T>,
): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()

  return {
    getState: () => state,

    setState: (updater: (prev: T) => T) => {
      const prev = state
      const next = updater(prev)
      if (Object.is(next, prev)) return  // 无变化则跳过
      state = next
      onChange?.({ newState: next, oldState: prev })
      for (const listener of listeners) listener()
    },

    subscribe: (listener: Listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)  // 返回取消订阅函数
    },
  }
}
```

### 设计特点

- **不可变更新**：`setState` 使用 updater 函数，确保状态不可变
- **浅比较优化**：`Object.is` 检查避免不必要的通知
- ** onChange 回调**：支持外部监听状态变化
- **订阅清理**：返回取消函数，方便组件卸载时清理

## AppState 类型

### 定义位置

[src/state/AppStateStore.ts](src/state/AppStateStore.ts)

### 核心结构

```typescript
export type AppState = DeepImmutable<{
  settings: SettingsJson             // 用户设置
  verbose: boolean                   // 详细模式
  mainLoopModel: ModelSetting        // 主循环模型
  mainLoopModelForSession: ModelSetting  // 会话级模型
  statusLineText: string | undefined // 状态栏文本
  expandedView: 'none' | 'tasks' | 'teammates'  // 展开视图
  isBriefOnly: boolean               // 简洁模式

  // 权限上下文
  toolPermissionContext: ToolPermissionContext

  // 模型与代理
  agent: string | undefined          // 代理名称
  kairosEnabled: boolean             // 助手模式启用

  // 远程会话
  remoteSessionUrl: string | undefined
  remoteConnectionStatus: 'connecting' | 'connected' | 'reconnecting' | 'disconnected'
  remoteBackgroundTaskCount: number

  // Always-on bridge 状态
  replBridgeEnabled: boolean
  replBridgeExplicit: boolean
  replBridgeOutboundOnly: boolean
  replBridgeConnected: boolean
  replBridgeSessionActive: boolean
  replBridgeReconnecting: boolean
  replBridgeConnectUrl: string | undefined
  replBridgeSessionUrl: string | undefined
  replBridgeEnvironmentId: string | undefined
  replBridgeSessionId: string | undefined
  replBridgeError: string | undefined
  replBridgeInitialName: string | undefined
  showRemoteCallout: boolean

  // MCP 状态
  mcp: {
    clients: MCPServerConnection[]
    tools: Tool[]
    commands: Command[]
    resources: Record<string, ServerResource[]>
    pluginReconnectKey: number
  }

  // 插件状态
  plugins: {
    enabled: LoadedPlugin[]
    disabled: LoadedPlugin[]
    commands: Command[]
    errors: PluginError[]
    installationStatus: { ... }
    needsRefresh: boolean
  }

  // 代理定义
  agentDefinitions: AgentDefinitionsResult
}> & {
  // 统一任务状态（不含 DeepImmutable，因 TaskState 有函数类型）
  tasks: { [taskId: string]: TaskState }

  // 代理名称注册表
  agentNameRegistry: Map<string, AgentId>

  // 前台任务 ID
  foregroundedTaskId?: string

  // 正在查看的代理任务 ID
  viewingAgentTaskId?: string

  // Companion 反应
  companionReaction?: string
  companionPetAt?: number

  // 文件历史
  fileHistory: FileHistoryState

  // 归属追踪
  attribution: AttributionState

  // 待办事项
  todos: { [agentId: string]: TodoList }

  // 远程代理任务建议
  remoteAgentTaskSuggestions: { summary: string; task: string }[]

  // 通知队列
  notifications: {
    current: Notification | null
    queue: Notification[]
  }

  // MCP Elicitation
  elicitation: {
    queue: ElicitationRequestEvent[]
  }

  // 思考模式
  thinkingEnabled: boolean | undefined

  // 提示建议
  promptSuggestionEnabled: boolean
  promptSuggestion: {
    text: string | null
    promptId: 'user_intent' | 'stated_intent' | null
    shownAt: number
    acceptedAt: number
    generationRequestId: string | null
  }

  // Speculation（预测执行）
  speculation: SpeculationState
  speculationSessionTimeSavedMs: number

  // 技能改进建议
  skillImprovement: {
    suggestion: { ... } | null
  }

  // 会话 Hooks
  sessionHooks: SessionHooksState

  // Tmux 集成（Tungsten）
  tungstenActiveSession?: { ... }
  tungstenLastCapturedTime?: number
  tungstenLastCommand?: { ... }
  tungstenPanelVisible?: boolean
  tungstenPanelAutoHidden?: boolean

  // WebBrowser 工具（Bagel）
  bagelActive?: boolean
  bagelUrl?: string
  bagelPanelVisible?: boolean

  // Computer Use MCP 状态
  computerUseMcpState?: { ... }

  // REPL 工具 VM 上下文
  replContext?: { ... }

  // 团队上下文（Swarm）
  teamContext?: { ... }

  // 独立代理上下文
  standaloneAgentContext?: { ... }

  // 消息收件箱
  inbox: {
    messages: Array<{ ... }>
  }

  // Worker 沙箱权限请求
  workerSandboxPermissions: { ... }
  pendingWorkerRequest: { ... } | null
  pendingSandboxRequest: { ... } | null

  // 认证版本
  authVersion: number

  // 初始消息
  initialMessage: {
    message: UserMessage
    clearContext?: boolean
    mode?: PermissionMode
    allowedPrompts?: AllowedPrompt[]
  } | null

  // Plan 验证状态
  pendingPlanVerification?: { ... }

  // 拒绝追踪
  denialTracking?: DenialTrackingState

  // 活动覆盖层
  activeOverlays: ReadonlySet<string>

  // 快速模式
  fastMode?: boolean

  // Advisor 模型
  advisorModel?: string

  // Effort 值
  effortValue?: EffortValue

  // Ultraplan 状态
  ultraplanLaunching?: boolean
  ultraplanSessionUrl?: string
  ultraplanPendingChoice?: { ... }
  ultraplanLaunchPending?: { ... }
  isUltraplanMode?: boolean

  // Bridge 权限回调
  replBridgePermissionCallbacks?: BridgePermissionCallbacks

  // Channel 权限回调
  channelPermissionCallbacks?: ChannelPermissionCallbacks
}
```

### Speculation 状态

预测执行（Speculation）是 Claude Code 的性能优化特性，在用户确认前预先执行可能的操作：

```typescript
export type SpeculationState =
  | { status: 'idle' }
  | {
      status: 'active'
      id: string
      abort: () => void
      startTime: number
      messagesRef: { current: Message[] }
      writtenPathsRef: { current: Set<string> }
      boundary: CompletionBoundary | null
      suggestionLength: number
      toolUseCount: number
      isPipelined: boolean
      contextRef: { current: REPLHookContext }
      pipelinedSuggestion?: { ... }
    }

export const IDLE_SPECULATION_STATE: SpeculationState = { status: 'idle' }
```

### CompletionBoundary

完成边界标记预测执行的结束点：

```typescript
export type CompletionBoundary =
  | { type: 'complete'; completedAt: number; outputTokens: number }
  | { type: 'bash'; command: string; completedAt: number }
  | { type: 'edit'; toolName: string; filePath: string; completedAt: number }
  | { type: 'denied_tool'; toolName: string; detail: string; completedAt: number }
```

## AppStateProvider

### 定义位置

[src/state/AppState.tsx](src/state/AppState.tsx)

### Provider 实现

```typescript
export const AppStoreContext = React.createContext<AppStateStore | null>(null)
const HasAppStateContext = React.createContext<boolean>(false)

type Props = {
  children: React.ReactNode
  initialState?: AppState
  onChangeAppState?: (args: { newState: AppState; oldState: AppState }) => void
}

export function AppStateProvider({ children, initialState, onChangeAppState }: Props) {
  // 防止嵌套 Provider
  const hasAppStateContext = useContext(HasAppStateContext)
  if (hasAppStateContext) {
    throw new Error("AppStateProvider can not be nested within another AppStateProvider")
  }

  // 创建 Store（只创建一次）
  const [store] = useState(() =>
    createStore(initialState ?? getDefaultAppState(), onChangeAppState)
  )

  // 检查是否应禁用 bypass permissions 模式
  useEffect(() => {
    const { toolPermissionContext } = store.getState()
    if (toolPermissionContext.isBypassPermissionsModeAvailable &&
        isBypassPermissionsModeDisabled()) {
      store.setState(prev => ({
        ...prev,
        toolPermissionContext: createDisabledBypassPermissionsContext(prev.toolPermissionContext)
      }))
    }
  }, [])

  // 监听外部设置变更
  const onSettingsChange = useEffectEvent((source: SettingSource) =>
    applySettingsChange(source, store.setState)
  )
  useSettingsChange(onSettingsChange)

  return (
    <HasAppStateContext.Provider value={true}>
      <AppStoreContext.Provider value={store}>
        <MailboxProvider>
          <VoiceProvider>{children}</VoiceProvider>
        </MailboxProvider>
      </AppStoreContext.Provider>
    </HasAppStateContext.Provider>
  )
}
```

### 设计特点

- **单例 Store**：使用 `useState` 确保 Store 只创建一次
- **嵌套保护**：`HasAppStateContext` 防止嵌套 Provider
- **设置同步**：`useSettingsChange` 监听外部文件变更
- **嵌套 Provider**：包含 `MailboxProvider` 和 `VoiceProvider`

## Hooks

### useAppState

订阅 AppState 的切片，仅在选中值变化时重渲染：

```typescript
export function useAppState<T>(selector: (state: AppState) => T): T {
  const store = useAppStore()

  const get = () => {
    const state = store.getState()
    const selected = selector(state)
    return selected
  }

  return useSyncExternalStore(store.subscribe, get, get)
}
```

**最佳实践**：

```typescript
// 好：多次调用独立字段
const verbose = useAppState(s => s.verbose)
const model = useAppState(s => s.mainLoopModel)

// 好：选择现有子对象引用
const { text, promptId } = useAppState(s => s.promptSuggestion)

// 坏：从 selector 返回新对象（Object.is 总是看到变化）
const data = useAppState(s => ({ text: s.promptSuggestion.text }))
```

### useSetAppState

获取更新函数，不订阅任何状态：

```typescript
export function useSetAppState(): (
  updater: (prev: AppState) => AppState
) => void {
  return useAppStore().setState
}
```

**特点**：
- 返回稳定引用，永不变化
- 使用此 hook 的组件不会因状态变化重渲染

### useAppStateStore

直接获取 Store（用于传递给非 React 代码）：

```typescript
export function useAppStateStore(): AppStateStore {
  return useAppStore()
}
```

### useAppStateMaybeOutsideOfProvider

安全版本，Provider 外返回 undefined：

```typescript
export function useAppStateMaybeOutsideOfProvider<T>(
  selector: (state: AppState) => T
): T | undefined {
  const store = useContext(AppStoreContext)
  return useSyncExternalStore(
    store ? store.subscribe : NOOP_SUBSCRIBE,
    () => store ? selector(store.getState()) : undefined
  )
}
```

## Selectors

### 定义位置

[src/state/selectors.ts](src/state/selectors.ts)

### getViewedTeammateTask

获取当前查看的队友任务：

```typescript
export function getViewedTeammateTask(
  appState: Pick<AppState, 'viewingAgentTaskId' | 'tasks'>
): InProcessTeammateTaskState | undefined {
  const { viewingAgentTaskId, tasks } = appState

  if (!viewingAgentTaskId) return undefined

  const task = tasks[viewingAgentTaskId]
  if (!task) return undefined

  if (!isInProcessTeammateTask(task)) return undefined

  return task
}
```

### getActiveAgentForInput

确定用户输入应路由到哪里：

```typescript
export type ActiveAgentForInput =
  | { type: 'leader' }
  | { type: 'viewed'; task: InProcessTeammateTaskState }
  | { type: 'named_agent'; task: LocalAgentTaskState }

export function getActiveAgentForInput(appState: AppState): ActiveAgentForInput {
  const viewedTask = getViewedTeammateTask(appState)
  if (viewedTask) {
    return { type: 'viewed', task: viewedTask }
  }

  const { viewingAgentTaskId, tasks } = appState
  if (viewingAgentTaskId) {
    const task = tasks[viewingAgentTaskId]
    if (task?.type === 'local_agent') {
      return { type: 'named_agent', task }
    }
  }

  return { type: 'leader' }
}
```

## 默认状态

### getDefaultAppState

```typescript
export function getDefaultAppState(): AppState {
  const initialMode: PermissionMode =
    isTeammate() && isPlanModeRequired() ? 'plan' : 'default'

  return {
    settings: getInitialSettings(),
    tasks: {},
    agentNameRegistry: new Map(),
    verbose: false,
    mainLoopModel: null,
    mainLoopModelForSession: null,
    statusLineText: undefined,
    expandedView: 'none',
    isBriefOnly: false,
    toolPermissionContext: {
      ...getEmptyToolPermissionContext(),
      mode: initialMode,
    },
    agent: undefined,
    agentDefinitions: { activeAgents: [], allAgents: [] },
    fileHistory: {
      snapshots: [],
      trackedFiles: new Set(),
      snapshotSequence: 0,
    },
    mcp: {
      clients: [],
      tools: [],
      commands: [],
      resources: {},
      pluginReconnectKey: 0,
    },
    plugins: {
      enabled: [],
      disabled: [],
      commands: [],
      errors: [],
      installationStatus: { marketplaces: [], plugins: [] },
      needsRefresh: false,
    },
    notifications: { current: null, queue: [] },
    elicitation: { queue: [] },
    thinkingEnabled: shouldEnableThinkingByDefault(),
    promptSuggestionEnabled: shouldEnablePromptSuggestion(),
    sessionHooks: new Map(),
    inbox: { messages: [] },
    speculation: IDLE_SPECULATION_STATE,
    speculationSessionTimeSavedMs: 0,
    authVersion: 0,
    initialMessage: null,
    activeOverlays: new Set(),
    fastMode: false,
    // ... 更多默认值
  }
}
```

## 状态更新流程

```
外部触发（设置变更、用户操作）
    ↓
store.setState(updater)
    ↓
Object.is 检查（无变化则跳过）
    ↓
更新内部 state
    ↓
onChange 回调（可选）
    ↓
通知所有 listeners
    ↓
useSyncExternalStore 触发重渲染
    ↓
组件按 selector 重新计算
    ↓
Object.is 比较（无变化则跳过重渲染）
```

## 状态分类

### 核心状态

| 字段 | 说明 |
|------|------|
| `settings` | 用户配置（JSON） |
| `toolPermissionContext` | 权限检查上下文 |
| `mainLoopModel` | 当前使用的模型 |
| `verbose` | 详细输出模式 |

### 任务状态

| 字段 | 说明 |
|------|------|
| `tasks` | 所有运行中的任务 |
| `foregroundedTaskId` | 前台任务 ID |
| `viewingAgentTaskId` | 正在查看的代理任务 |
| `agentNameRegistry` | 代理名称 → ID 映射 |

### 连接状态

| 字段 | 说明 |
|------|------|
| `mcp.clients` | MCP 服务器连接 |
| `mcp.tools` | MCP 提供的工具 |
| `plugins.enabled/disabled` | 插件状态 |
| `remoteConnectionStatus` | 远程会话连接状态 |
| `replBridge*` | Always-on bridge 状态 |

### UI 状态

| 字段 | 说明 |
|------|------|
| `expandedView` | 展开的视图面板 |
| `footerSelection` | 底部栏焦点项 |
| `notifications` | 通知队列 |
| `promptSuggestion` | 提示建议 |
| `activeOverlays` | 活动覆盖层（对话框） |

### 特性状态

| 字段 | 说明 |
|------|------|
| `speculation` | 预测执行状态 |
| `thinkingEnabled` | 思考模式 |
| `fastMode` | 快速模式 |
| `computerUseMcpState` | Computer Use MCP |
| `teamContext` | Swarm 团队上下文 |

## 关键文件

- [src/state/store.ts](src/state/store.ts) — Store 工厂函数
- [src/state/AppStateStore.ts](src/state/AppStateStore.ts) — AppState 类型定义
- [src/state/AppState.tsx](src/state/AppState.tsx) — Provider 与 Hooks
- [src/state/selectors.ts](src/state/selectors.ts) — 状态选择器
- [src/state/onChangeAppState.ts](src/state/onChangeAppState.ts) — 变更回调处理
- [src/state/teammateViewHelpers.ts](src/state/teammateViewHelpers.ts) — 队友视图辅助

## 关联文档

- [04-查询引擎](04-query-engine.md)
- [07-权限系统](07-permission-system.md)
- [08-MCP集成](08-mcp-integration.md)