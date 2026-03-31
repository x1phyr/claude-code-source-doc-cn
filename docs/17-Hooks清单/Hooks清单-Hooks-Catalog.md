# Hooks 清单

## 概述

Hooks 是 React 函数组件的状态管理基础。Claude Code 有 80+ 个自定义 hooks，位于 `src/hooks/` 目录。

## Hooks 目录结构

```
src/hooks/
├── useAppState.tsx        # 状态订阅
├── useCanUseTool.tsx      # 权限检查
├── useMergedTools.tsx     # 工具合并
├── useMergedCommands.tsx  # 命令合并
├── useSettingsChange.ts   # 设置变更
├── useReplBridge.tsx      # Bridge 状态
├── useMainLoopModel.tsx   # 模型选择
├── useTerminalSize.tsx    # 终端尺寸
├── useKeybinding.tsx      # 快捷键
├── useTheme.ts            # 主题
└── ...                    # 更多 hooks
```

## 状态管理 Hooks

### useAppState

订阅 AppState 切片：

```typescript
export function useAppState<T>(selector: (state: AppState) => T): T {
  const store = useAppStore()
  const get = () => selector(store.getState())
  return useSyncExternalStore(store.subscribe, get, get)
}

// 用法
const verbose = useAppState(s => s.verbose)
const model = useAppState(s => s.mainLoopModel)
```

### useSetAppState

获取状态更新函数：

```typescript
export function useSetAppState() {
  return useAppStore().setState
}

// 用法
const setState = useSetAppState()
setState(prev => ({ ...prev, verbose: true }))
```

### useAppStateStore

获取完整 Store：

```typescript
export function useAppStateStore(): AppStateStore {
  return useAppStore()
}

// 用于传递给非 React 代码
const store = useAppStateStore()
someExternalApi.setStore(store)
```

## 工具相关 Hooks

### useMergedTools

合并内置工具与 MCP 工具：

```typescript
export function useMergedTools(): Tools {
  const permissionContext = useAppState(s => s.toolPermissionContext)
  const mcpTools = useAppState(s => s.mcp.tools)

  return useMemo(() => {
    return assembleToolPool(permissionContext, mcpTools)
  }, [permissionContext, mcpTools])
}
```

### useMergedCommands

合并所有命令来源：

```typescript
export function useMergedCommands(): Command[] {
  const [commands, setCommands] = useState<Command[]>([])

  useEffect(() => {
    getCommands(getCwd()).then(setCommands)
  }, [])

  return commands
}
```

### useCanUseTool

权限检查函数：

```typescript
export type CanUseToolFn = (
  tool: Tool,
  input: Record<string, unknown>,
  context: ToolUseContext,
  assistantMessage: AssistantMessage,
  toolUseID: string,
  forceDecision?: boolean
) => Promise<PermissionResult>

export function useCanUseTool(): CanUseToolFn {
  const permissionContext = useAppState(s => s.toolPermissionContext)

  return useCallback(async (tool, input, context, message, id, force) => {
    // 权限检查逻辑
    return checkPermissions(tool, input, permissionContext)
  }, [permissionContext])
}
```

## 设置相关 Hooks

### useSettingsChange

监听设置文件变更：

```typescript
export function useSettingsChange(
  callback: (source: SettingSource) => void
) {
  useEffect(() => {
    const unsubscribe = subscribeToSettingsChanges(callback)
    return unsubscribe
  }, [callback])
}
```

### useSettings

获取当前设置：

```typescript
export function useSettings(): SettingsJson {
  return useAppState(s => s.settings)
}
```

## 模型相关 Hooks

### useMainLoopModel

获取主循环模型：

```typescript
export function useMainLoopModel(): ModelSetting {
  return useAppState(s => s.mainLoopModel)
}
```

### useModelOptions

获取模型选项：

```typescript
export function useModelOptions(): ModelOption[] {
  const settings = useSettings()
  return useMemo(() => {
    return getAvailableModels(settings)
  }, [settings])
}
```

## MCP 相关 Hooks

### useManageMCPConnections

管理 MCP 连接：

```typescript
export function useManageMCPConnections() {
  const setAppState = useSetAppState()

  useEffect(() => {
    // 连接所有配置的 MCP 服务器
    const connections = connectAllServers()

    setAppState(prev => ({
      ...prev,
      mcp: {
        ...prev.mcp,
        clients: connections.clients,
        tools: connections.tools,
        commands: connections.commands,
        resources: connections.resources
      }
    }))

    return () => {
      // 清理连接
      connections.cleanup()
    }
  }, [])
}
```

### useReplBridge

Bridge 状态管理：

```typescript
export function useReplBridge() {
  const enabled = useAppState(s => s.replBridgeEnabled)
  const connected = useAppState(s => s.replBridgeConnected)
  const sessionActive = useAppState(s => s.replBridgeSessionActive)

  return {
    enabled,
    connected,
    sessionActive,
    toggle: () => { /* ... */ },
    connect: () => { /* ... */ },
    disconnect: () => { /* ... */ }
  }
}
```

## UI 相关 Hooks

### useTerminalSize

终端尺寸：

```typescript
export function useTerminalSize(): { columns: number; rows: number } {
  const [size, setSize] = useState(() => ({
    columns: process.stdout.columns || 80,
    rows: process.stdout.rows || 24
  }))

  useEffect(() => {
    const handler = () => {
      setSize({
        columns: process.stdout.columns || 80,
        rows: process.stdout.rows || 24
      })
    }

    process.stdout.on('resize', handler)
    return () => process.stdout.off('resize', handler)
  }, [])

  return size
}
```

### useTheme

当前主题：

```typescript
export function useTheme(): Theme {
  const settings = useSettings()
  return useMemo(() => getTheme(settings.theme), [settings.theme])
}
```

### useKeybinding

快捷键注册：

```typescript
export function useKeybinding(
  key: string,
  callback: () => void,
  deps: unknown[] = []
) {
  useEffect(() => {
    const handler = (event: KeyEvent) => {
      if (event.key === key) {
        callback()
      }
    }

    stdin.on('key', handler)
    return () => stdin.off('key', handler)
  }, [key, ...deps])
}
```

## 任务相关 Hooks

### useTasks

任务列表：

```typescript
export function useTasks(): TaskState[] {
  const tasks = useAppState(s => s.tasks)
  return useMemo(() => Object.values(tasks), [tasks])
}
```

### useForegroundTask

前台任务：

```typescript
export function useForegroundTask(): TaskState | undefined {
  const tasks = useAppState(s => s.tasks)
  const foregroundId = useAppState(s => s.foregroundedTaskId)

  return foregroundId ? tasks[foregroundId] : undefined
}
```

## 会话相关 Hooks

### useSession

会话状态：

```typescript
export function useSession() {
  const messages = useAppState(s => s.messages)
  const sessionId = useAppState(s => s.sessionId)

  return {
    messages,
    sessionId,
    clear: () => { /* ... */ },
    export: () => { /* ... */ }
  }
}
```

### useMessages

消息列表：

```typescript
export function useMessages(): Message[] {
  return useAppState(s => s.messages)
}
```

## 记忆相关 Hooks

### useMemory

记忆系统：

```typescript
export function useMemory() {
  const memories = useAppState(s => s.memories)

  return {
    memories,
    add: (memory: Memory) => { /* ... */ },
    remove: (id: string) => { /* ... */ }
  }
}
```

## 动画 Hooks

### useAnimation

动画帧：

```typescript
export function useAnimation(frames: number, interval: number = 100): number {
  const [frame, setFrame] = useState(0)

  useEffect(() => {
    const timer = setInterval(() => {
      setFrame(f => (f + 1) % frames)
    }, interval)

    return () => clearInterval(timer)
  }, [frames, interval])

  return frame
}
```

## 其他 Hooks

### useDebounce

防抖值：

```typescript
export function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value)

  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value)
    }, delay)

    return () => clearTimeout(timer)
  }, [value, delay])

  return debouncedValue
}
```

### useThrottle

节流值：

```typescript
export function useThrottle<T>(value: T, limit: number): T {
  const [throttledValue, setThrottledValue] = useState(value)
  const lastRan = useRef(Date.now())

  useEffect(() => {
    const handler = setTimeout(() => {
      if (Date.now() - lastRan.current >= limit) {
        setThrottledValue(value)
        lastRan.current = Date.now()
      }
    }, limit - (Date.now() - lastRan.current))

    return () => clearTimeout(handler)
  }, [value, limit])

  return throttledValue
}
```

### usePrevious

前一个值：

```typescript
export function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T>()

  useEffect(() => {
    ref.current = value
  }, [value])

  return ref.current
}
```

## Hook 设计模式

### 状态订阅优化

```typescript
// 好：订阅独立字段
const verbose = useAppState(s => s.verbose)
const model = useAppState(s => s.mainLoopModel)

// 坏：返回新对象
const { verbose, model } = useAppState(s => ({
  verbose: s.verbose,
  model: s.mainLoopModel
}))
```

### 使用 useCallback 稳定引用

```typescript
export function useCanUseTool(): CanUseToolFn {
  const permissionContext = useAppState(s => s.toolPermissionContext)

  return useCallback(async (tool, input) => {
    return checkPermissions(tool, input, permissionContext)
  }, [permissionContext])
}
```

### 使用 useMemo 缓存计算

```typescript
export function useMergedTools(): Tools {
  const permissionContext = useAppState(s => s.toolPermissionContext)
  const mcpTools = useAppState(s => s.mcp.tools)

  return useMemo(() => {
    return assembleToolPool(permissionContext, mcpTools)
  }, [permissionContext, mcpTools])
}
```

## 关键文件

- [src/state/AppState.tsx](src/state/AppState.tsx) — 状态 Hooks
- [src/hooks/useCanUseTool.tsx](src/hooks/useCanUseTool.tsx) — 权限 Hook
- [src/hooks/useMergedTools.tsx](src/hooks/useMergedTools.tsx) — 工具合并
- [src/hooks/useSettingsChange.ts](src/hooks/useSettingsChange.ts) — 设置变更

## 关联文档

- [09-状态管理](09-state-management.md)
- [07-权限系统](07-permission-system.md)
- [13-UI组件详解](13-components-detail.md)