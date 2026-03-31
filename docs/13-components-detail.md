# UI 组件详解

## 概述

Claude Code 使用 Ink 框架（React for CLI）构建终端用户界面。组件位于 `src/components/` 目录，负责渲染、交互和状态展示。

## 组件目录结构

```
src/components/
├── App.tsx                 # 应用根组件
├── Message.tsx             # 消息渲染
├── LogoV2/                 # Logo 与欢迎界面
├── HelpV2/                 # 帮助界面
├── PromptInput/            # 输入组件
├── CustomSelect/           # 自定义选择器
├── FeedbackSurvey/         # 反馈调查
├── Markdown.tsx            # Markdown 渲染
├── HighlightedCode.tsx     # 代码高亮
├── ToolUseMessage/         # 工具调用消息
├── PermissionDialog/       # 权限对话框
├── ConfigDialog/           # 配置对话框
└── ...                     # 更多组件
```

## 核心组件

### App.tsx

应用根组件，协调整体布局：

```typescript
export function App(props: AppProps): React.ReactNode {
  return (
    <AppStateProvider>
      <ErrorBoundary>
        <REPL />
      </ErrorBoundary>
    </AppStateProvider>
  )
}
```

### REPL.tsx

主 REPL 组件，管理会话循环：

```typescript
export function REPL() {
  const messages = useAppState(s => s.messages)
  const input = useAppState(s => s.input)

  return (
    <Box flexDirection="column">
      <MessageList messages={messages} />
      <PromptInput />
    </Box>
  )
}
```

### Message.tsx

消息渲染组件：

```typescript
export function Message({ message }: { message: Message }) {
  switch (message.role) {
    case 'user':
      return <UserMessage message={message} />
    case 'assistant':
      return <AssistantMessage message={message} />
    default:
      return null
  }
}
```

## LogoV2 组件

### 目录结构

[src/components/LogoV2/](src/components/LogoV2/)

```
LogoV2/
├── AnimatedAsterisk.tsx    # 动画星号
├── AnimatedClawd.tsx       # 动画 Clawd
├── ChannelsNotice.tsx      # 渠道通知
├── Clawd.tsx               # Clawd 角色
├── CondensedLogo.tsx       # 精简 Logo
├── EmergencyTip.tsx        # 紧急提示
├── Feed.tsx                # 信息流
├── FeedColumn.tsx          # 信息流列
├── GuestPassesUpsell.tsx   # 访客通行证
├── LogoV2.tsx              # Logo 主组件
├── OverageCreditUpsell.tsx # 超额信用
├── VoiceModeNotice.tsx     # 语音模式通知
├── WelcomeV2.tsx           # 欢迎界面
└── feedConfigs.tsx         # 信息流配置
```

### WelcomeV2

欢迎界面：

```typescript
export function WelcomeV2() {
  return (
    <Box flexDirection="column">
      <LogoV2 />
      <Feed />
      <PromptSuggestion />
    </Box>
  )
}
```

### AnimatedClawd

动画 Clawd 角色：

```typescript
export function AnimatedClawd() {
  const frame = useAnimation(ANIMATION_FRAMES)
  return <Text>{CLAWD_FRAMES[frame]}</Text>
}
```

## HelpV2 组件

### 目录结构

[src/components/HelpV2/](src/components/HelpV2/)

```
HelpV2/
├── Commands.tsx            # 命令列表
├── General.tsx             # 通用帮助
└── HelpV2.tsx              # 帮助主组件
```

### HelpV2

```typescript
export function HelpV2() {
  const [tab, setTab] = useState('general')

  return (
    <Box flexDirection="column">
      <TabBar active={tab} onChange={setTab} />
      {tab === 'general' && <General />}
      {tab === 'commands' && <Commands />}
    </Box>
  )
}
```

## PromptInput 组件

### 目录结构

[src/components/PromptInput/](src/components/PromptInput/)

```
PromptInput/
├── PromptInput.tsx         # 输入主组件
├── PromptInputFooter.tsx   # 底部栏
├── Suggestions.tsx         # 建议
└── ...
```

### PromptInput

主输入组件：

```typescript
export function PromptInput() {
  const [value, setValue] = useState('')
  const onSubmit = useSetAppState(s => s.submitMessage)

  return (
    <Box>
      <TextInput
        value={value}
        onChange={setValue}
        onSubmit={() => onSubmit(value)}
      />
      <PromptInputFooter />
    </Box>
  )
}
```

## 对话框组件

### 权限对话框

| 组件 | 说明 |
|------|------|
| BypassPermissionsModeDialog | 绕过权限模式确认 |
| MCPServerApprovalDialog | MCP 服务器批准 |
| PermissionDialog | 通用权限对话框 |

### 配置对话框

| 组件 | 说明 |
|------|------|
| ConfigDialog | 配置管理 |
| ModelDialog | 模型选择 |
| ThemeDialog | 主题选择 |

### 其他对话框

| 组件 | 说明 |
|------|------|
| ExportDialog | 导出会话 |
| FeedbackSurvey | 反馈调查 |
| IdleReturnDialog | 空闲返回确认 |
| InvalidConfigDialog | 无效配置警告 |

## 消息渲染组件

### ToolUseMessage

工具调用消息渲染：

```typescript
export function ToolUseMessage({ toolUse }: { toolUse: ToolUse }) {
  const tool = getToolByName(toolUse.name)

  return (
    <Box>
      <ToolUseHeader tool={tool} />
      <ToolUseContent input={toolUse.input} />
    </Box>
  )
}
```

### HighlightedCode

代码高亮渲染：

```typescript
export function HighlightedCode({
  code,
  language
}: {
  code: string
  language: string
}) {
  const tokens = useHighlight(code, language)

  return (
    <Text>
      {tokens.map((token, i) => (
        <Text key={i} color={token.color}>{token.content}</Text>
      ))}
    </Text>
  )
}
```

### Markdown

Markdown 渲染：

```typescript
export function Markdown({ children }: { children: string }) {
  const ast = parseMarkdown(children)

  return (
    <Box flexDirection="column">
      {ast.map((node, i) => (
        <MarkdownNode key={i} node={node} />
      ))}
    </Box>
  )
}
```

## 选择器组件

### CustomSelect

自定义选择器：

```typescript
export function Select({
  options,
  value,
  onChange
}: SelectProps) {
  return (
    <Box flexDirection="column">
      {options.map((option, i) => (
        <SelectOption
          key={i}
          option={option}
          selected={value === option.value}
          onSelect={() => onChange(option.value)}
        />
      ))}
    </Box>
  )
}
```

### SelectMulti

多选选择器：

```typescript
export function SelectMulti({
  options,
  selected,
  onChange
}: SelectMultiProps) {
  return (
    <Box flexDirection="column">
      {options.map((option, i) => (
        <SelectOption
          key={i}
          option={option}
          selected={selected.has(option.value)}
          onSelect={() => toggle(option.value)}
        />
      ))}
    </Box>
  )
}
```

## 状态指示器组件

### MemoryUsageIndicator

内存使用指示器：

```typescript
export function MemoryUsageIndicator() {
  const usage = useAppState(s => s.memoryUsage)

  return (
    <Text color={usage > 80 ? 'red' : 'green'}>
      Memory: {usage}%
    </Text>
  )
}
```

### IdeStatusIndicator

IDE 状态指示器：

```typescript
export function IdeStatusIndicator() {
  const connected = useAppState(s => s.ideConnected)

  return (
    <Text color={connected ? 'green' : 'yellow'}>
      {connected ? '●' : '○'} IDE
    </Text>
  )
}
```

## 布局组件

### FullscreenLayout

全屏布局：

```typescript
export function FullscreenLayout({ children }: { children: React.ReactNode }) {
  return (
    <Box
      flexDirection="column"
      width="100%"
      height="100%"
    >
      {children}
    </Box>
  )
}
```

### MessageRow

消息行：

```typescript
export function MessageRow({ children }: { children: React.ReactNode }) {
  return (
    <Box marginBottom={1}>
      {children}
    </Box>
  )
}
```

## 组件设计模式

### 使用 Hooks 提取状态

```typescript
// 自定义 Hook
function useMessageList() {
  const messages = useAppState(s => s.messages)
  const loading = useAppState(s => s.loading)
  return { messages, loading }
}

// 组件
function MessageList() {
  const { messages, loading } = useMessageList()
  // ...
}
```

### 条件渲染

```typescript
function Message({ message }: { message: Message }) {
  if (message.type === 'user') {
    return <UserMessage message={message} />
  }

  if (message.type === 'assistant') {
    return <AssistantMessage message={message} />
  }

  return null
}
```

### 渲染函数分离

```typescript
function ToolUseMessage({ toolUse }: Props) {
  return renderToolUse(toolUse)
}

function renderToolUse(toolUse: ToolUse) {
  switch (toolUse.name) {
    case 'Read':
      return <ReadToolMessage input={toolUse.input} />
    case 'Edit':
      return <EditToolMessage input={toolUse.input} />
    default:
      return <DefaultToolMessage toolUse={toolUse} />
  }
}
```

## 关键文件

- [src/components/App.tsx](src/components/App.tsx) — 应用根组件
- [src/components/Message.tsx](src/components/Message.tsx) — 消息渲染
- [src/components/LogoV2/LogoV2.tsx](src/components/LogoV2/LogoV2.tsx) — Logo 组件
- [src/components/HelpV2/HelpV2.tsx](src/components/HelpV2/HelpV2.tsx) — 帮助组件
- [src/components/Markdown.tsx](src/components/Markdown.tsx) — Markdown 渲染

## 关联文档

- [17-响应式系统](17-reactive-system.md)
- [09-状态管理](09-state-management.md)
- [14-Hooks清单](14-hooks-catalog.md)