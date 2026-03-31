# 核心组件

## 概述

Claude Code 使用 Ink 框架（React for CLI）构建终端用户界面。核心组件负责渲染、交互和状态管理。

## 核心文件

- [src/components/App.tsx](src/components/App.tsx) — 主应用组件
- [src/components/TextInput.tsx](src/components/TextInput.tsx) — 文本输入
- [src/components/Message.tsx](src/components/Message.tsx) — 消息渲染
- [src/components/StatusLine.tsx](src/components/StatusLine.tsx) — 状态栏

## 组件架构

```
App
├── FullscreenLayout
│   ├── Messages (消息列表)
│   │   └── MessageRow[]
│   │       └── Message
│   │           ├── UserMessage
│   │           ├── AssistantMessage
│   │           └── ToolResultMessage
│   ├── StatusLine (状态栏)
│   └── PromptInput (输入区)
│       ├── TextInput
│       ├── PromptInputFooter
│       └── Notifications
```

## 主应用组件

### App.tsx

```typescript
export function App({
  debug,
  verbose,
  ...props
}: AppProps): React.ReactElement {
  const appState = useAppState()
  const setAppState = useSetAppState()

  // 主循环模型
  const mainLoopModel = useMainLoopModel()

  // 处理消息提交
  const handleSubmit = useCallback(async (input, helpers) => {
    await handlePromptSubmit(input, {
      context: getToolUseContext(),
      setAppState,
      helpers,
    })
  }, [getToolUseContext, setAppState])

  return (
    <FullscreenLayout>
      {/* 消息列表 */}
      <Messages
        messages={appState.messages}
        verbose={verbose}
      />

      {/* 状态栏 */}
      <StatusLine
        model={mainLoopModel}
        isLoading={appState.isProcessing}
      />

      {/* 输入区 */}
      <PromptInput
        input={appState.userInput}
        onInputChange={setUserInput}
        onSubmit={handleSubmit}
        isLoading={appState.isProcessing}
      />
    </FullscreenLayout>
  )
}
```

## 文本输入组件

### TextInput.tsx

```typescript
export default function TextInput({
  value,
  onChange,
  onSubmit,
  focus = true,
  mask,
  multiline = false,
  cursorChar = '█',
  highlightPastedText,
  invert,
  themeText,
  columns,
  ...props
}: TextInputProps): React.ReactElement {
  const { onInput, renderedValue, offset, cursorLine } = useTextInput({
    value,
    onChange,
    onSubmit,
    multiline,
    cursorChar,
    columns,
    ...
  })

  useInput(
    (input, key) => onInput(input, key),
    { isActive: focus },
  )

  return (
    <Box flexDirection="column">
      <Text>
        {renderedValue}
      </Text>
    </Box>
  )
}
```

### BaseTextInput.tsx

```typescript
export function BaseTextInput({
  value,
  cursorOffset,
  ...props
}: BaseTextInputProps): React.ReactElement {
  // 渲染前半部分
  const beforeCursor = value.slice(0, cursorOffset)
  // 渲染光标字符
  const cursorChar = value[cursorOffset] ?? ' '
  // 渲染后半部分
  const afterCursor = value.slice(cursorOffset + 1)

  return (
    <Text>
      {beforeCursor}
      <Text inverse>{cursorChar}</Text>
      {afterCursor}
    </Text>
  )
}
```

## 消息组件

### Message.tsx

```typescript
export function Message({
  message,
  verbose,
  ...props
}: MessageProps): React.ReactElement | null {
  switch (message.type) {
    case 'user':
      return <UserMessage message={message} />

    case 'assistant':
      return (
        <AssistantMessage
          message={message}
          verbose={verbose}
        />
      )

    case 'attachment':
      return <AttachmentMessage attachment={message.attachment} />

    case 'system':
      return <SystemMessage message={message} />

    case 'progress':
      return <ProgressMessage progress={message.progress} />

    default:
      return null
  }
}
```

### AssistantMessage

```typescript
export function AssistantMessage({
  message,
  verbose,
}: AssistantMessageProps): React.ReactElement {
  const content = message.message.content

  return (
    <Box flexDirection="column">
      {Array.isArray(content) ? (
        content.map((block, index) => (
          <ContentBlock
            key={index}
            block={block}
            verbose={verbose}
          />
        ))
      ) : (
        <Markdown content={content} />
      )}
    </Box>
  )
}

function ContentBlock({ block, verbose }: ContentBlockProps) {
  switch (block.type) {
    case 'text':
      return <Markdown content={block.text} />

    case 'thinking':
      return verbose ? (
        <ThinkingBlock content={block.thinking} />
      ) : null

    case 'tool_use':
      return <ToolUseBlock block={block} />

    default:
      return null
  }
}
```

### UserMessage

```typescript
export function UserMessage({
  message,
}: UserMessageProps): React.ReactElement {
  const content = message.message.content

  return (
    <Box flexDirection="column" borderStyle="round" borderColor="cyan">
      {Array.isArray(content) ? (
        content.map((block, index) => (
          <UserContentBlock key={index} block={block} />
        ))
      ) : (
        <Text>{content}</Text>
      )}
    </Box>
  )
}
```

## 工具消息组件

### ToolResultMessage

```typescript
export function ToolResultMessage({
  toolName,
  result,
  verbose,
}: ToolResultMessageProps): React.ReactElement {
  if (result.is_error) {
    return <ToolErrorMessage toolName={toolName} result={result} />
  }

  switch (toolName) {
    case 'Read':
      return <FileReadResult result={result} />

    case 'Edit':
    case 'Write':
      return <FileEditResult result={result} />

    case 'Bash':
      return <BashResult result={result} verbose={verbose} />

    case 'Glob':
    case 'Grep':
      return <SearchResult result={result} />

    default:
      return <GenericToolResult result={result} />
  }
}
```

### BashResult

```typescript
export function BashResult({
  result,
  verbose,
}: BashResultProps): React.ReactElement {
  const [isExpanded, setIsExpanded] = useState(false)
  const output = result.output

  return (
    <Box flexDirection="column">
      <Box>
        <Text dimColor>{output.summary ?? 'Command executed'}</Text>
      </Box>

      {verbose && (
        <Box>
          <Text
            dimColor
            onPress={() => setIsExpanded(!isExpanded)}
          >
            {isExpanded ? '▼ Collapse' : '▶ Expand'}
          </Text>
        </Box>
      )}

      {isExpanded && (
        <Box flexDirection="column" marginLeft={2}>
          {output.stdout && (
            <Text dimColor>{output.stdout}</Text>
          )}
          {output.stderr && (
            <Text color="red">{output.stderr}</Text>
          )}
        </Box>
      )}
    </Box>
  )
}
```

## 状态栏组件

### StatusLine.tsx

```typescript
export function StatusLine({
  model,
  isLoading,
  ...props
}: StatusLineProps): React.ReactElement {
  return (
    <Box flexDirection="row" justifyContent="space-between">
      {/* 左侧：模型信息 */}
      <Box>
        <Text dimColor>
          {modelDisplayString(model)}
          {isLoading && ' ●'}
        </Text>
      </Box>

      {/* 中间：统计 */}
      <Box>
        <Text dimColor>
          {formatTokens(tokensUsed)} tokens
        </Text>
      </Box>

      {/* 右侧：快捷键提示 */}
      <Box>
        <Text dimColor>
          Ctrl+C to exit
        </Text>
      </Box>
    </Box>
  )
}
```

## Markdown 组件

### Markdown.tsx

```typescript
export function Markdown({
  content,
  ...props
}: MarkdownProps): React.ReactElement {
  const tokens = useMemo(() => marked.lexer(content), [content])

  return (
    <Box flexDirection="column">
      {tokens.map((token, index) => (
        <MarkdownToken key={index} token={token} />
      ))}
    </Box>
  )
}

function MarkdownToken({ token }: TokenProps): React.ReactElement {
  switch (token.type) {
    case 'heading':
      return (
        <Text bold={token.depth <= 2}>
          {token.text}
        </Text>
      )

    case 'paragraph':
      return <Text>{token.text}</Text>

    case 'code':
      return (
        <HighlightedCode
          code={token.text}
          language={token.lang}
        />
      )

    case 'list':
      return (
        <Box flexDirection="column">
          {token.items.map((item, i) => (
            <Text key={i}>• {item.text}</Text>
          ))}
        </Box>
      )

    default:
      return <Text>{token.raw}</Text>
  }
}
```

## 代码高亮组件

### HighlightedCode.tsx

```typescript
export function HighlightedCode({
  code,
  language,
}: HighlightedCodeProps): React.ReactElement {
  const highlighted = useMemo(() => {
    try {
      return highlight(code, language)
    } catch {
      return code
    }
  }, [code, language])

  return (
    <Box flexDirection="column" borderStyle="round">
      {language && (
        <Box>
          <Text dimColor>{language}</Text>
        </Box>
      )}
      <Text>
        {highlighted}
      </Text>
    </Box>
  )
}
```

## Spinner 组件

### Spinner.tsx

```typescript
const FRAMES = ['⠋', '⠙', '⠹', '⠸', '⠼', '⠴', '⠦', '⠧', '⠇', '⠏']

export function Spinner({
  text,
}: SpinnerProps): React.ReactElement {
  const [frame, setFrame] = useState(0)

  useEffect(() => {
    const timer = setInterval(() => {
      setFrame(f => (f + 1) % FRAMES.length)
    }, 80)

    return () => clearInterval(timer)
  }, [])

  return (
    <Text>
      <Text color="cyan">{FRAMES[frame]}</Text>
      {text && <Text> {text}</Text>}
    </Text>
  )
}
```

## 对话框组件

### ConfirmDialog

```typescript
export function ConfirmDialog({
  message,
  onConfirm,
  onCancel,
}: ConfirmDialogProps): React.ReactElement {
  const [selected, setSelected] = useState<'yes' | 'no'>('no')

  useInput((input, key) => {
    if (key.leftArrow) setSelected('no')
    if (key.rightArrow) setSelected('yes')
    if (key.return) {
      selected === 'yes' ? onConfirm() : onCancel()
    }
  })

  return (
    <Box flexDirection="column" borderStyle="double">
      <Text>{message}</Text>
      <Box>
        <Text
          bold={selected === 'no'}
          inverse={selected === 'no'}
        >
          [No]
        </Text>
        <Text> </Text>
        <Text
          bold={selected === 'yes'}
          inverse={selected === 'yes'}
        >
          [Yes]
        </Text>
      </Box>
    </Box>
  )
}
```

## 通知组件

### Notifications.tsx

```typescript
export function Notifications(): React.ReactElement | null {
  const notifications = useNotifications()

  if (notifications.length === 0) return null

  return (
    <Box flexDirection="column">
      {notifications.map(notification => (
        <Notification
          key={notification.key}
          notification={notification}
        />
      ))}
    </Box>
  )
}

function Notification({
  notification,
}: NotificationProps): React.ReactElement {
  return (
    <Box>
      <Text dimColor inverse>
        {notification.text}
      </Text>
    </Box>
  )
}
```

## 关键文件

- [src/components/App.tsx](src/components/App.tsx) — 主应用
- [src/components/TextInput.tsx](src/components/TextInput.tsx) — 文本输入
- [src/components/Message.tsx](src/components/Message.tsx) — 消息渲染
- [src/components/StatusLine.tsx](src/components/StatusLine.tsx) — 状态栏
- [src/components/Markdown.tsx](src/components/Markdown.tsx) — Markdown 渲染
- [src/components/Spinner.tsx](src/components/Spinner.tsx) — 加载动画

## 关联文档

- [15-UI组件/UI组件详解-Components-Detail.md](UI组件详解-Components-Detail.md)
- [15-UI组件/终端渲染-Terminal-Rendering.md](终端渲染-Terminal-Rendering.md)
- [16-响应式系统/响应式系统详解-Reactive-System.md](../16-响应式系统/响应式系统详解-Reactive-System.md)