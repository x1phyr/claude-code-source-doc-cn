# 响应式系统详解

## 概述

Claude Code 使用 Ink 框架构建终端用户界面。Ink 是 React for CLI，允许使用 React 组件模型开发命令行应用。

## Ink 框架原理

### 核心概念

Ink 将 React 组件渲染到终端：

1. **React 组件**：使用 JSX 编写 UI
2. **Yoga 布局引擎**：Flexbox 布局
3. **终端渲染**：将组件输出为 ANSI 转义序列

### 渲染流程

```
React 组件树
    ↓
Yoga 布局计算
    ↓
ANSI 转义序列
    ↓
终端输出
```

## 组件渲染

### 基本组件

```tsx
import { Box, Text } from 'ink'

function MyComponent() {
  return (
    <Box flexDirection="column">
      <Text color="green">Hello, World!</Text>
    </Box>
  )
}
```

### 布局组件

```tsx
<Box flexDirection="column">    // 垂直布局
<Box flexDirection="row">       // 水平布局
<Box justifyContent="center">   // 居中
<Box alignItems="center">       // 对齐
<Box padding={1}>               // 内边距
<Box margin={1}>                // 外边距
<Box width="100%">              // 宽度
<Box height="100%">             // 高度
```

### 文本样式

```tsx
<Text color="green">Green text</Text>
<Text bold>Bold text</Text>
<Text italic>Italic text</Text>
<Text underline>Underlined text</Text>
<Text dimColor>Dimmed text</Text>
<Text inverse>Inverted colors</Text>
```

## 状态更新

### useInput 钩子

```tsx
import { useInput } from 'ink'

function InputComponent() {
  useInput((input, key) => {
    if (key.return) {
      // 处理回车
    }
    if (key.escape) {
      // 处理 ESC
    }
    if (key.upArrow) {
      // 处理上箭头
    }
  })
}
```

### 状态订阅

```tsx
function MessageList() {
  const messages = useAppState(s => s.messages)

  return (
    <Box flexDirection="column">
      {messages.map(msg => (
        <Message key={msg.id} message={msg} />
      ))}
    </Box>
  )
}
```

### 重渲染机制

```
setState 调用
    ↓
通知 subscribers
    ↓
useSyncExternalStore 触发
    ↓
组件重渲染
    ↓
Ink diff 并更新终端
```

## 键盘事件处理

### 全局输入处理

```tsx
useInput((input, key) => {
  // 普通按键
  if (input === 'q') {
    process.exit(0)
  }

  // 特殊按键
  if (key.return) { /* 回车 */ }
  if (key.escape) { /* ESC */ }
  if (key.upArrow) { /* 上 */ }
  if (key.downArrow) { /* 下 */ }
  if (key.leftArrow) { /* 左 */ }
  if (key.rightArrow) { /* 右 */ }
  if (key.tab) { /* Tab */ }
  if (key.backspace) { /* 退格 */ }
  if (key.delete) { /* 删除 */ }
  if (key.ctrl && input === 'c') { /* Ctrl+C */ }
})
```

### 快捷键注册

```tsx
export function useKeybinding(
  key: string,
  callback: () => void
) {
  useInput((input, k) => {
    if (input === key || k[key + 'Arrow']) {
      callback()
    }
  })
}
```

## 终端尺寸适配

### 获取尺寸

```tsx
import { useStdout } from 'ink'

function ResponsiveComponent() {
  const { stdout } = useStdout()

  return (
    <Box width={stdout.columns} height={stdout.rows}>
      {/* 内容 */}
    </Box>
  )
}
```

### 响应式布局

```tsx
function ResponsiveLayout() {
  const { stdout } = useStdout()
  const isNarrow = stdout.columns < 80

  return (
    <Box flexDirection={isNarrow ? 'column' : 'row'}>
      {/* 根据宽度调整布局 */}
    </Box>
  )
}
```

## 动画与过渡

### 帧动画

```tsx
function AnimatedComponent() {
  const [frame, setFrame] = useState(0)

  useEffect(() => {
    const timer = setInterval(() => {
      setFrame(f => (f + 1) % FRAMES.length)
    }, 100)
    return () => clearInterval(timer)
  }, [])

  return <Text>{FRAMES[frame]}</Text>
}
```

### 加载动画

```tsx
function Spinner() {
  const frames = ['⠋', '⠙', '⠹', '⠸', '⠼', '⠴', '⠦', '⠧', '⠇', '⠏']
  const [frame, setFrame] = useState(0)

  useEffect(() => {
    const timer = setInterval(() => {
      setFrame(f => (f + 1) % frames.length)
    }, 80)
    return () => clearInterval(timer)
  }, [])

  return <Text color="cyan">{frames[frame]}</Text>
}
```

## 应用入口

### 渲染应用

```tsx
import { render } from 'ink'

function main() {
  const { unmount, waitUntilExit } = render(<App />)

  // 等待退出
  waitUntilExit().then(() => {
    console.log('App exited')
  })
}
```

### 退出处理

```tsx
function App() {
  useInput((input, key) => {
    if (key.escape || (key.ctrl && input === 'c')) {
      process.exit(0)
    }
  })

  return <MainUI />
}
```

## 性能优化

### 避免不必要的重渲染

```tsx
// 使用 selector 订阅特定状态
const name = useAppState(s => s.name)  // 好

// 而不是订阅整个状态
const state = useAppState(s => s)      // 坏
```

### 使用 React.memo

```tsx
const MessageItem = React.memo(function MessageItem({ message }) {
  return <Text>{message.content}</Text>
})
```

### 使用 useMemo/useCallback

```tsx
function MessageList({ messages }) {
  const sortedMessages = useMemo(
    () => messages.sort((a, b) => a.timestamp - b.timestamp),
    [messages]
  )

  return sortedMessages.map(msg => <MessageItem key={msg.id} message={msg} />)
}
```

## 样式系统

### 主题

```tsx
function useTheme() {
  const settings = useSettings()
  return themes[settings.theme]
}

function StyledText({ children }) {
  const theme = useTheme()
  return <Text color={theme.primary}>{children}</Text>
}
```

### 颜色常量

```typescript
const colors = {
  primary: 'cyan',
  secondary: 'magenta',
  success: 'green',
  warning: 'yellow',
  error: 'red',
  muted: 'gray'
}
```

## 调试

### React DevTools

Ink 支持 React DevTools：

```tsx
import { render } from 'ink'

render(<App />, {
  debug: true,
  exitOnCtrlC: false
})
```

### 日志输出

```tsx
// 使用 console.error 输出调试信息（不影响渲染）
console.error('Debug:', someValue)
```

## 关键文件

- [src/components/App.tsx](src/components/App.tsx) — 应用根组件
- [src/main.tsx](src/main.tsx) — 入口点
- [src/hooks/](src/hooks/) — 自定义 Hooks

## 关联文档

- [13-UI组件详解](13-components-detail.md)
- [14-Hooks清单](14-hooks-catalog.md)
- [09-状态管理](09-state-management.md)