# 动画效果

## 概述

Claude Code 使用终端动画增强用户体验，包括加载动画、进度指示和过渡效果。

## 核心文件

- [src/components/Spinner.tsx](src/components/Spinner.tsx) — 加载动画
- [src/components/AgentProgressLine.tsx](src/components/AgentProgressLine.tsx) — 进度条
- [src/utils/animations.ts](src/utils/animations.ts) — 动画工具

## 动画类型

### 加载动画

```typescript
const SPINNER_FRAMES = ['⠋', '⠙', '⠹', '⠸', '⠼', '⠴', '⠦', '⠧', '⠇', '⠏']

export function Spinner({ text }: SpinnerProps): React.ReactElement {
  const [frame, setFrame] = useState(0)

  useEffect(() => {
    const timer = setInterval(() => {
      setFrame(f => (f + 1) % SPINNER_FRAMES.length)
    }, 80)

    return () => clearInterval(timer)
  }, [])

  return (
    <Text>
      <Text color="cyan">{SPINNER_FRAMES[frame]}</Text>
      {text && <Text> {text}</Text>}
    </Text>
  )
}
```

### 进度条

```typescript
export function ProgressBar({
  progress,
  width = 20,
}: ProgressBarProps): React.ReactElement {
  const filled = Math.round(width * progress)
  const empty = width - filled

  return (
    <Text>
      <Text color="green">{'█'.repeat(filled)}</Text>
      <Text dimColor>{'░'.repeat(empty)}</Text>
      <Text> {Math.round(progress * 100)}%</Text>
    </Text>
  )
}
```

### 脉冲效果

```typescript
export function Pulse({
  children,
  color,
}: PulseProps): React.ReactElement {
  const [opacity, setOpacity] = useState(1)

  useEffect(() => {
    let increasing = false
    const timer = setInterval(() => {
      setOpacity(prev => {
        if (prev <= 0.3) increasing = true
        if (prev >= 1) increasing = false
        return increasing ? prev + 0.1 : prev - 0.1
      })
    }, 50)

    return () => clearInterval(timer)
  }, [])

  return (
    <Text color={color} dimColor={opacity < 0.5}>
      {children}
    </Text>
  )
}
```

## Agent 进度动画

### 进度线

```typescript
export function AgentProgressLine({
  agentId,
  description,
  status,
}: AgentProgressLineProps): React.ReactElement | null {
  const color = useAgentColor(agentId)

  if (status === 'pending') {
    return <Text dimColor>○ {description}</Text>
  }

  if (status === 'running') {
    return (
      <Text>
        <Text color={color}>●</Text>
        <Text> {description}</Text>
        <Spinner text="..." />
      </Text>
    )
  }

  if (status === 'completed') {
    return <Text color="green">✓ {description}</Text>
  }

  return <Text color="red">✗ {description}</Text>
}
```

### 状态转换动画

```typescript
export function useStateAnimation(
  status: AgentStatus,
): { isAnimating: boolean, animatingClass: string } {
  const [isAnimating, setIsAnimating] = useState(false)

  useEffect(() => {
    if (status === 'running') {
      setIsAnimating(true)
    } else {
      // 延迟停止动画，显示完成状态
      const timer = setTimeout(() => setIsAnimating(false), 500)
      return () => clearTimeout(timer)
    }
  }, [status])

  return {
    isAnimating,
    animatingClass: isAnimating ? 'animating' : '',
  }
}
```

## Thinking 动画

### 思考指示器

```typescript
const THINKING_FRAMES = [
  '·   ',
  ' ·  ',
  '  · ',
  '   ·',
]

export function ThinkingIndicator(): React.ReactElement {
  const [frame, setFrame] = useState(0)

  useEffect(() => {
    const timer = setInterval(() => {
      setFrame(f => (f + 1) % THINKING_FRAMES.length)
    }, 200)

    return () => clearInterval(timer)
  }, [])

  return (
    <Text dimColor>
      Thinking{THINKING_FRAMES[frame]}
    </Text>
  )
}
```

## 颜色渐变

### 彩虹效果

```typescript
const RAINBOW_COLORS = [
  'red', 'yellow', 'green', 'cyan', 'blue', 'magenta',
]

export function useRainbowColor(interval = 200): string {
  const [index, setIndex] = useState(0)

  useEffect(() => {
    const timer = setInterval(() => {
      setIndex(i => (i + 1) % RAINBOW_COLORS.length)
    }, interval)

    return () => clearInterval(timer)
  }, [interval])

  return RAINBOW_COLORS[index]
}

export function RainbowText({ children }: RainbowTextProps): React.ReactElement {
  const color = useRainbowColor()
  return <Text color={color}>{children}</Text>
}
```

## 打字机效果

### 逐字显示

```typescript
export function Typewriter({
  text,
  speed = 50,
  onComplete,
}: TypewriterProps): React.ReactElement {
  const [displayed, setDisplayed] = useState('')

  useEffect(() => {
    let index = 0
    const timer = setInterval(() => {
      if (index < text.length) {
        setDisplayed(text.slice(0, index + 1))
        index++
      } else {
        clearInterval(timer)
        onComplete?.()
      }
    }, speed)

    return () => clearInterval(timer)
  }, [text, speed])

  return <Text>{displayed}</Text>
}
```

## 过渡动画

### 淡入效果

```typescript
export function FadeIn({
  children,
  duration = 300,
}: FadeInProps): React.ReactElement {
  const [visible, setVisible] = useState(false)

  useEffect(() => {
    const timer = setTimeout(() => setVisible(true), 50)
    return () => clearTimeout(timer)
  }, [])

  return (
    <Text dimColor={!visible}>
      {children}
    </Text>
  )
}
```

### 滑入效果

```typescript
export function SlideIn({
  children,
  direction = 'left',
}: SlideInProps): React.ReactElement {
  const [offset, setOffset] = useState(direction === 'left' ? -10 : 10)

  useEffect(() => {
    const timer = setTimeout(() => setOffset(0), 50)
    return () => clearTimeout(timer)
  }, [])

  // 通过空格模拟偏移
  const padding = offset > 0 ? ' '.repeat(offset) : ''

  return (
    <Text>
      {padding}{children}
    </Text>
  )
}
```

## 计数动画

### 数字滚动

```typescript
export function CountUp({
  end,
  duration = 1000,
}: CountUpProps): React.ReactElement {
  const [current, setCurrent] = useState(0)

  useEffect(() => {
    const steps = 20
    const increment = end / steps
    const interval = duration / steps

    let count = 0
    const timer = setInterval(() => {
      count += increment
      if (count >= end) {
        setCurrent(end)
        clearInterval(timer)
      } else {
        setCurrent(Math.floor(count))
      }
    }, interval)

    return () => clearInterval(timer)
  }, [end, duration])

  return <Text>{current.toLocaleString()}</Text>
}
```

## 性能考虑

### 帧率限制

```typescript
const FPS = 30
const FRAME_INTERVAL = 1000 / FPS

export function useAnimationFrame(callback: () => void): void {
  useEffect(() => {
    let lastTime = 0
    let rafId: number

    const animate = (time: number) => {
      if (time - lastTime >= FRAME_INTERVAL) {
        callback()
        lastTime = time
      }
      rafId = requestAnimationFrame(animate)
    }

    rafId = requestAnimationFrame(animate)
    return () => cancelAnimationFrame(rafId)
  }, [callback])
}
```

### 动画合并

```typescript
// 使用单个动画循环更新多个元素
export function useAnimationGroup(): {
  register: (id: string, update: () => void) => void
  unregister: (id: string) => void
} {
  const updates = useRef(new Map<string, () => void>())

  useEffect(() => {
    const timer = setInterval(() => {
      for (const update of updates.current.values()) {
        update()
      }
    }, 80)

    return () => clearInterval(timer)
  }, [])

  return {
    register: (id, update) => updates.current.set(id, update),
    unregister: (id) => updates.current.delete(id),
  }
}
```

## 关键文件

- [src/components/Spinner.tsx](src/components/Spinner.tsx) — 加载动画
- [src/components/AgentProgressLine.tsx](src/components/AgentProgressLine.tsx) — 进度线
- [src/components/ThinkingIndicator.tsx](src/components/ThinkingIndicator.tsx) — 思考指示

## 关联文档

- [15-UI组件/核心组件-Core-Components.md](核心组件-Core-Components.md)
- [15-UI组件/终端渲染-Terminal-Rendering.md](终端渲染-Terminal-Rendering.md)