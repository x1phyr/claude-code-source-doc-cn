# 终端渲染

## 概述

Claude Code 使用 Ink 框架在终端中渲染 React 组件。本文档介绍终端渲染的实现细节，包括布局、样式和性能优化。

## 核心文件

- [src/ink/index.ts](src/ink/index.ts) — Ink 导出
- [src/ink/render.tsx](src/ink/render.tsx) — 渲染函数
- [src/ink/stringWidth.ts](src/ink/stringWidth.ts) — 字符串宽度计算

## 渲染架构

```
React 组件树
    ↓
Ink reconciler
    ↓
Yoga 布局引擎
    ↓
ANSI 转义序列
    ↓
终端输出
```

## Ink 框架集成

### 核心 API

```typescript
import { Box, Text, render, useInput, useApp } from 'src/ink.js'

// 渲染应用
const { unmount, waitUntilExit } = render(<App />)

// 等待退出
await waitUntilExit()
unmount()
```

### Box 组件

```typescript
<Box
  flexDirection="row" | "column"          // 布局方向
  justifyContent="flex-start" | "center" | "flex-end" | "space-between"  // 主轴对齐
  alignItems="flex-start" | "center" | "flex-end"  // 交叉轴对齐
  width={number | string}                 // 宽度
  height={number | string}                // 高度
  minWidth={number}                       // 最小宽度
  maxWidth={number}                       // 最大宽度
  padding={number}                        // 内边距
  margin={number}                         // 外边距
  borderStyle="single" | "double" | "round"  // 边框样式
  borderColor="red" | "blue" | ...        // 边框颜色
>
  {children}
</Box>
```

### Text 组件

```typescript
<Text
  color="red" | "green" | "blue" | ...    // 文本颜色
  backgroundColor="black" | "white" | ... // 背景颜色
  bold                                    // 粗体
  italic                                  // 斜体
  underline                               // 下划线
  strikethrough                           // 删除线
  inverse                                 // 反色
  dimColor                                // 暗淡
  wrap="wrap" | "truncate" | "truncate-end"  // 换行方式
>
  {text}
</Text>
```

## 字符串宽度计算

### CJK 字符处理

```typescript
import stringWidth from 'src/ink/stringWidth.js'

// 计算字符串显示宽度
const width = stringWidth('你好世界')  // 返回 8（每个中文字符占 2）

// ASCII 字符
const width2 = stringWidth('hello')  // 返回 5
```

### 宽字符对齐

```typescript
function padRight(text: string, width: number): string {
  const currentWidth = stringWidth(text)
  const padding = width - currentWidth
  return text + ' '.repeat(Math.max(0, padding))
}

function padLeft(text: string, width: number): string {
  const currentWidth = stringWidth(text)
  const padding = width - currentWidth
  return ' '.repeat(Math.max(0, padding)) + text
}
```

## 边框渲染

### 边框字符

```typescript
const BORDER_CHARS = {
  single: {
    topLeft: '┌',
    topRight: '┐',
    bottomLeft: '└',
    bottomRight: '┘',
    horizontal: '─',
    vertical: '│',
  },
  double: {
    topLeft: '╔',
    topRight: '╗',
    bottomLeft: '╚',
    bottomRight: '╝',
    horizontal: '═',
    vertical: '║',
  },
  round: {
    topLeft: '╭',
    topRight: '╮',
    bottomLeft: '╰',
    bottomRight: '╯',
    horizontal: '─',
    vertical: '│',
  },
}
```

### 边框渲染函数

```typescript
function renderBorder(
  content: string[],
  style: 'single' | 'double' | 'round',
  width: number,
): string[] {
  const chars = BORDER_CHARS[style]
  const lines: string[] = []

  // 顶边框
  lines.push(
    chars.topLeft + chars.horizontal.repeat(width - 2) + chars.topRight
  )

  // 内容行
  for (const line of content) {
    lines.push(
      chars.vertical + padRight(line, width - 2) + chars.vertical
    )
  }

  // 底边框
  lines.push(
    chars.bottomLeft + chars.horizontal.repeat(width - 2) + chars.bottomRight
  )

  return lines
}
```

## ANSI 转义序列

### 颜色代码

```typescript
const COLORS = {
  black: '\x1b[30m',
  red: '\x1b[31m',
  green: '\x1b[32m',
  yellow: '\x1b[33m',
  blue: '\x1b[34m',
  magenta: '\x1b[35m',
  cyan: '\x1b[36m',
  white: '\x1b[37m',
  reset: '\x1b[0m',
}

function colorize(text: string, color: keyof typeof COLORS): string {
  return `${COLORS[color]}${text}${COLORS.reset}`
}
```

### 样式代码

```typescript
const STYLES = {
  bold: '\x1b[1m',
  dim: '\x1b[2m',
  italic: '\x1b[3m',
  underline: '\x1b[4m',
  inverse: '\x1b[7m',
  strikethrough: '\x1b[9m',
  reset: '\x1b[0m',
}
```

### 光标控制

```typescript
const CURSOR = {
  hide: '\x1b[?25l',
  show: '\x1b[?25h',
  home: '\x1b[H',
  clearScreen: '\x1b[2J',
  clearLine: '\x1b[2K',
  moveUp: (n: number) => `\x1b[${n}A`,
  moveDown: (n: number) => `\x1b[${n}B`,
  moveRight: (n: number) => `\x1b[${n}C`,
  moveLeft: (n: number) => `\x1b[${n}D`,
}
```

## 全屏模式

### 全屏布局

```typescript
import { useTerminalSize } from 'src/hooks/useTerminalSize.js'

function FullscreenLayout({ children }: Props): React.ReactElement {
  const { columns, rows } = useTerminalSize()

  return (
    <Box
      flexDirection="column"
      width={columns}
      height={rows}
    >
      {children}
    </Box>
  )
}
```

### 终端大小监听

```typescript
export function useTerminalSize(): { columns: number, rows: number } {
  const [size, setSize] = useState({
    columns: process.stdout.columns,
    rows: process.stdout.rows,
  })

  useEffect(() => {
    const onResize = () => {
      setSize({
        columns: process.stdout.columns,
        rows: process.stdout.rows,
      })
    }

    process.stdout.on('resize', onResize)
    return () => {
      process.stdout.off('resize', onResize)
    }
  }, [])

  return size
}
```

## 滚动实现

### 消息滚动

```typescript
function ScrollableMessages({
  messages,
  maxVisible,
}: ScrollableMessagesProps): React.ReactElement {
  const [scrollOffset, setScrollOffset] = useState(0)
  const { rows } = useTerminalSize()

  const visibleMessages = messages.slice(
    Math.max(0, messages.length - maxVisible - scrollOffset),
    messages.length - scrollOffset,
  )

  useInput((input, key) => {
    if (key.upArrow) {
      setScrollOffset(Math.min(messages.length - maxVisible, scrollOffset + 1))
    }
    if (key.downArrow) {
      setScrollOffset(Math.max(0, scrollOffset - 1))
    }
  })

  return (
    <Box flexDirection="column">
      {scrollOffset > 0 && (
        <Text dimColor>↑ {scrollOffset} more messages</Text>
      )}
      {visibleMessages.map(msg => (
        <Message key={msg.id} message={msg} />
      ))}
    </Box>
  )
}
```

## 性能优化

### 虚拟滚动

```typescript
function VirtualList({
  items,
  itemHeight,
  containerHeight,
}: VirtualListProps): React.ReactElement {
  const [scrollTop, setScrollTop] = useState(0)

  const startIndex = Math.floor(scrollTop / itemHeight)
  const endIndex = Math.min(
    items.length,
    startIndex + Math.ceil(containerHeight / itemHeight) + 1,
  )

  const visibleItems = items.slice(startIndex, endIndex)

  return (
    <Box flexDirection="column" height={containerHeight}>
      {visibleItems.map((item, index) => (
        <Box key={startIndex + index} height={itemHeight}>
          {item}
        </Box>
      ))}
    </Box>
  )
}
```

### 增量更新

```typescript
// 只更新变化的部分
function useIncrementalRender(
  content: string,
): string {
  const prevContentRef = useRef(content)

  const diff = useMemo(() => {
    if (prevContentRef.current === content) return null
    prevContentRef.current = content
    return content
  }, [content])

  return diff
}
```

## 备用屏幕缓冲

### 启用备用屏幕

```typescript
const ALT_SCREEN = {
  enter: '\x1b[?1049h',
  exit: '\x1b[?1049l',
}

function enterAltScreen(): void {
  process.stdout.write(ALT_SCREEN.enter)
}

function exitAltScreen(): void {
  process.stdout.write(ALT_SCREEN.exit)
}

// 在应用生命周期中
useEffect(() => {
  enterAltScreen()
  return () => exitAltScreen()
}, [])
```

## 鼠标支持

### 启用鼠标事件

```typescript
const MOUSE = {
  enable: '\x1b[?1000h',   // 基本鼠标报告
  disable: '\x1b[?1000l',
  sgr: '\x1b[?1006h',      // SGR 扩展模式
}

function useMouseEvents(): void {
  useEffect(() => {
    process.stdout.write(MOUSE.enable)
    process.stdout.write(MOUSE.sgr)

    return () => {
      process.stdout.write(MOUSE.disable)
    }
  }, [])
}
```

## 终端兼容性

### 功能检测

```typescript
function detectTerminalCapabilities(): {
  hasColors: boolean
  hasUtf8: boolean
  hasTrueColor: boolean
  hasAltScreen: boolean
} {
  const term = process.env.TERM ?? ''

  return {
    hasColors: process.stdout.isTTY && process.stdout.hasColors(),
    hasUtf8: term.includes('xterm') || term.includes('screen'),
    hasTrueColor: term.includes('truecolor') || term.includes('24bit'),
    hasAltScreen: term.includes('xterm') || term.includes('screen'),
  }
}
```

### 降级处理

```typescript
function renderWithFallback(
  component: React.ReactElement,
  capabilities: TerminalCapabilities,
): string {
  if (!capabilities.hasUtf8) {
    // 使用 ASCII 边框
    return renderWithAscii(component)
  }

  if (!capabilities.hasColors) {
    // 移除颜色
    return renderWithoutColors(component)
  }

  return render(component)
}
```

## 关键文件

- [src/ink/index.ts](src/ink/index.ts) — Ink 导出
- [src/ink/stringWidth.ts](src/ink/stringWidth.ts) — 字符宽度
- [src/hooks/useTerminalSize.ts](src/hooks/useTerminalSize.ts) — 终端尺寸

## 关联文档

- [15-UI组件/核心组件-Core-Components.md](核心组件-Core-Components.md)
- [15-UI组件/动画效果-Animation-Effects.md](动画效果-Animation-Effects.md)
- [16-响应式系统/响应式系统详解-Reactive-System.md](../16-响应式系统/响应式系统详解-Reactive-System.md)