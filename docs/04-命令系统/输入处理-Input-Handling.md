# 输入处理

## 概述

输入处理是 Claude Code 用户交互的核心，负责接收、解析和处理用户在终端中的所有输入，包括文本编辑、键盘快捷键、斜杠命令和特殊模式切换。

## 核心文件

- [src/hooks/useTextInput.ts](src/hooks/useTextInput.ts) — 文本输入处理 Hook
- [src/utils/processUserInput/processUserInput.ts](src/utils/processUserInput/processUserInput.ts) — 用户输入处理
- [src/components/PromptInput/PromptInput.tsx](src/components/PromptInput/PromptInput.tsx) — 输入组件

## 输入流程

```
用户输入 → useInput (Ink)
    ↓
useTextInput Hook
    ├── 键盘映射
    ├── 光标操作
    └── Kill/Yank
    ↓
processUserInput
    ├── 斜杠命令检测
    ├── 附件加载
    └── 消息创建
    ↓
QueryEngine
```

## 文本输入 Hook

### useTextInput

```typescript
export function useTextInput({
  value,
  onChange,
  onSubmit,
  onExit,
  onHistoryUp,
  onHistoryDown,
  multiline,
  cursorChar,
  columns,
  ...
}: UseTextInputProps): TextInputState {
  // 创建光标对象
  const cursor = Cursor.fromText(value, columns, offset)

  // 键盘处理器映射
  const handleCtrl = mapInput([
    ['a', () => cursor.startOfLine()],     // 行首
    ['b', () => cursor.left()],             // 左移
    ['c', handleCtrlC],                     // 中断
    ['d', handleCtrlD],                     // 删除/退出
    ['e', () => cursor.endOfLine()],        // 行尾
    ['f', () => cursor.right()],            // 右移
    ['k', killToLineEnd],                   // 删除到行尾
    ['u', killToLineStart],                 // 删除到行首
    ['w', killWordBefore],                  // 删除前一词
    ['y', yank],                            // 粘贴
  ])

  const handleMeta = mapInput([
    ['b', () => cursor.prevWord()],         // 前一词
    ['f', () => cursor.nextWord()],         // 后一词
    ['d', () => cursor.deleteWordAfter()],  // 删除后一词
    ['y', handleYankPop],                   // Yank Pop
  ])

  // 主输入处理
  function onInput(input: string, key: Key): void {
    const nextCursor = mapKey(key)(input)
    if (nextCursor && !cursor.equals(nextCursor)) {
      onChange(nextCursor.text)
      setOffset(nextCursor.offset)
    }
  }

  return { onInput, renderedValue, offset, ... }
}
```

### 光标操作

```typescript
export class Cursor {
  text: string
  offset: number
  columns: number

  // 移动操作
  left(): Cursor
  right(): Cursor
  startOfLine(): Cursor
  endOfLine(): Cursor
  prevWord(): Cursor
  nextWord(): Cursor
  up(): Cursor
  down(): Cursor

  // 编辑操作
  insert(text: string): Cursor
  backspace(): Cursor
  del(): Cursor
  deleteWordBefore(): Cursor
  deleteWordAfter(): Cursor
  deleteToLineEnd(): { cursor: Cursor, killed: string }
  deleteToLineStart(): { cursor: Cursor, killed: string }
}
```

## Kill/Yank 系统

类似 Emacs 的 Kill Ring：

```typescript
// Kill Ring 存储
const killRing: string[] = []
const yankState = { lastYankStart: 0, lastYankLength: 0 }

function pushToKillRing(text: string, mode: 'append' | 'prepend'): void {
  if (mode === 'append' && killRing.length > 0) {
    killRing[killRing.length - 1] += text
  } else if (mode === 'prepend' && killRing.length > 0) {
    killRing[killRing.length - 1] = text + killRing[killRing.length - 1]
  } else {
    killRing.push(text)
  }
}

function getLastKill(): string {
  return killRing[killRing.length - 1] ?? ''
}

function yank(): Cursor {
  const text = getLastKill()
  if (text.length > 0) {
    const newCursor = cursor.insert(text)
    recordYank(cursor.offset, text.length)
    return newCursor
  }
  return cursor
}

function yankPop(): { text: string, start: number, length: number } | null {
  // 循环 Kill Ring
  if (killRing.length > 1) {
    killRing.unshift(killRing.pop()!)
    return { text: getLastKill(), ... }
  }
  return null
}
```

## 双击检测

用于安全退出和清空输入：

```typescript
export function useDoublePress(
  onFirstPress: (show: boolean) => void,
  onSecondPress: () => void,
  onCancel?: () => void,
): () => void {
  let timeout: Timer | null = null

  return () => {
    if (timeout) {
      clearTimeout(timeout)
      timeout = null
      onSecondPress()
    } else {
      onFirstPress(true)
      timeout = setTimeout(() => {
        timeout = null
        onFirstPress(false)
        onCancel?.()
      }, 1000)  // 1 秒窗口
    }
  }
}

// 使用示例：双击 Ctrl+C 退出
const handleCtrlC = useDoublePress(
  show => onExitMessage?.(show, 'Ctrl-C'),
  () => onExit?.(),
  () => onChange(''),  // 单击清空输入
)
```

## 输入模式

### 模式类型

```typescript
export type PromptInputMode =
  | 'prompt'     // 正常提示模式
  | 'bash'       // Bash 命令模式 (!command)
  | 'vim'        // Vim 编辑模式

export function getModeFromInput(input: string): PromptInputMode {
  if (input.startsWith('!')) return 'bash'
  return 'prompt'
}
```

### Bash 模式

```typescript
// !command 进入 Bash 模式
if (inputString !== null && mode === 'bash') {
  const { processBashCommand } = await import('./processBashCommand.js')
  return await processBashCommand(
    inputString.slice(1),  // 去掉 ! 前缀
    precedingInputBlocks,
    attachmentMessages,
    context,
    setToolJSX,
  )
}
```

## 用户输入处理

### processUserInput

```typescript
export async function processUserInput({
  input,
  mode,
  setToolJSX,
  context,
  pastedContents,
  ideSelection,
  messages,
  querySource,
  canUseTool,
  skipSlashCommands,
  bridgeOrigin,
  isMeta,
}: ProcessUserInputOptions): Promise<ProcessUserInputBaseResult> {
  // 1. 处理图像粘贴
  const imageContents = pastedContents
    ? Object.values(pastedContents).filter(isValidImagePaste)
    : []

  const imageContentBlocks = await processPastedImages(imageContents)

  // 2. Bridge 安全命令检查
  let effectiveSkipSlash = skipSlashCommands
  if (bridgeOrigin && inputString?.startsWith('/')) {
    const cmd = findCommand(parsed.commandName, context.options.commands)
    if (cmd && isBridgeSafeCommand(cmd)) {
      effectiveSkipSlash = false  // 允许安全命令
    }
  }

  // 3. Ultraplan 关键词检测
  if (hasUltraplanKeyword(inputString)) {
    const rewritten = replaceUltraplanKeyword(inputString)
    return await processSlashCommand(`/ultraplan ${rewritten}`, ...)
  }

  // 4. 加载附件消息
  const attachmentMessages = await getAttachmentMessages(
    inputString,
    context,
    ideSelection,
    [],
    messages,
    querySource,
  )

  // 5. 斜杠命令处理
  if (inputString?.startsWith('/') && !effectiveSkipSlash) {
    return await processSlashCommand(inputString, ...)
  }

  // 6. 普通文本提示
  return processTextPrompt(normalizedInput, ...)
}
```

## 附件加载

### getAttachmentMessages

```typescript
export async function* getAttachmentMessages(
  input: string,
  context: ToolUseContext,
  ideSelection: IDESelection | null,
  queuedCommands: QueuedCommand[],
  messages?: Message[],
  querySource?: QuerySource,
): AsyncGenerator<AttachmentMessage> {
  // IDE 选择上下文
  if (ideSelection) {
    yield createAttachmentMessage({
      type: 'ide_selection',
      content: ideSelection.text,
      filePath: ideSelection.filePath,
      startLine: ideSelection.startLine,
      endLine: ideSelection.endLine,
    })
  }

  // @agent 提及
  const agentMentions = extractAgentMentions(input)
  for (const mention of agentMentions) {
    yield createAttachmentMessage({
      type: 'agent_mention',
      agentType: mention.agentType,
    })
  }

  // 文件引用
  const fileRefs = extractFileReferences(input)
  for (const ref of fileRefs) {
    const content = await readFile(ref.path)
    yield createAttachmentMessage({
      type: 'file_content',
      content,
      filePath: ref.path,
    })
  }
}
```

## Hook 集成

### UserPromptSubmit Hook

```typescript
// 执行用户提交 Hook
for await (const hookResult of executeUserPromptSubmitHooks(
  inputMessage,
  permissionMode,
  context,
)) {
  // 阻断性错误
  if (hookResult.blockingError) {
    return {
      messages: [createSystemMessage(hookResult.blockingError, 'warning')],
      shouldQuery: false,
    }
  }

  // 阻止继续
  if (hookResult.preventContinuation) {
    result.messages.push(createUserMessage({ content: 'Stopped by hook' }))
    result.shouldQuery = false
    return result
  }

  // 额外上下文
  if (hookResult.additionalContexts) {
    result.messages.push(createAttachmentMessage({
      type: 'hook_additional_context',
      content: hookResult.additionalContexts,
    }))
  }
}
```

## 特殊输入处理

### SSH 环境适配

```typescript
// SSH/tmux 环境的 DEL 字符处理
if (!key.backspace && !key.delete && input.includes('\x7f')) {
  const delCount = (input.match(/\x7f/g) || []).length
  let currentCursor = cursor
  for (let i = 0; i < delCount; i++) {
    currentCursor = currentCursor.deleteTokenBefore() ?? currentCursor.backspace()
  }
  onChange(currentCursor.text)
  return
}
```

### SSH 回车合并

```typescript
// SSH 慢速连接中 "o\r" 合并处理
if (filteredInput.length > 1 && filteredInput.endsWith('\r')) {
  // 用户输入 + Enter 被合并为一个输入块
  onSubmit?.(nextCursor.text)
}
```

## Vim 模式

### Vim 输入处理

```typescript
export function useVimInput({
  value,
  onChange,
  mode,
  setMode,
}: VimInputProps): VimInputState {
  // Normal 模式命令
  const normalCommands = {
    'i': () => setMode('insert'),
    'a': () => { cursor.right(); setMode('insert') },
    'h': () => cursor.left(),
    'j': () => cursor.down(),
    'k': () => cursor.up(),
    'l': () => cursor.right(),
    'x': () => cursor.del(),
    'dd': () => { onChange(''); setOffset(0) },
  }

  // Insert 模式使用标准 useTextInput
  if (mode === 'insert') {
    return useTextInput(...)
  }

  // Normal 模式处理
  function onVimInput(input: string, key: Key): void {
    const cmd = pendingCommand + input
    if (normalCommands[cmd]) {
      normalCommands[cmd]()
      setPendingCommand('')
    } else if (Object.keys(normalCommands).some(c => c.startsWith(cmd))) {
      setPendingCommand(cmd)  // 等待更多输入
    } else {
      setPendingCommand('')
    }
  }

  return { onInput: onVimInput, mode, ... }
}
```

## 键绑定系统

### Keybinding Context

```typescript
// 键绑定上下文优先级
const KEYBINDING_CONTEXTS = [
  'autocomplete',    // 自动补全激活
  'history_search',  // 历史搜索
  'global_search',   // 全局搜索
  'prompt',          // 默认提示
]

// 上下文特定的键绑定
useKeybindings({
  context: 'prompt',
  bindings: {
    'enter': { action: 'submit' },
    'up': { action: 'history_up' },
    'down': { action: 'history_down' },
    'ctrl+l': { action: 'clear_screen' },
  },
})
```

## 终端兼容性

### Apple Terminal 特殊处理

```typescript
// Apple Terminal 不支持 Shift+Enter 键绑定
if (env.terminal === 'Apple_Terminal') {
  // 使用原生 macOS modifier 检测
  if (isModifierPressed('shift')) {
    return cursor.insert('\n')  // 多行
  }
}
```

### Modifier 预热

```typescript
// Apple Terminal 需要 prewarm modifiers module
if (env.terminal === 'Apple_Terminal') {
  prewarmModifiers()
}
```

## 统计日志

```typescript
logEvent('tengu_input_mode_change', {
  from_mode: previousMode,
  to_mode: newMode,
})

logEvent('tengu_pasted_image_resize_attempt', {
  original_size_bytes: pastedImage.content.length,
})

logEvent('tengu_subagent_at_mention', {
  is_subagent_only: trimmedInput === '@agent-...',
  is_prefix: trimmedInput.startsWith('@agent-'),
})
```

## 关键文件

- [src/hooks/useTextInput.ts](src/hooks/useTextInput.ts) — 文本输入 Hook
- [src/utils/Cursor.ts](src/utils/Cursor.ts) — 光标操作
- [src/utils/processUserInput/processUserInput.ts](src/utils/processUserInput/processUserInput.ts) — 输入处理
- [src/components/PromptInput/PromptInput.tsx](src/components/PromptInput/PromptInput.tsx) — 输入组件
- [src/hooks/useDoublePress.ts](src/hooks/useDoublePress.ts) — 双击检测

## 关联文档

- [04-命令系统/命令系统详解-Command-System.md](命令系统详解-Command-System.md)
- [15-UI组件/核心组件-Core-Components.md](../15-UI组件/核心组件-Core-Components.md)
- [17-Hooks清单/Hooks清单-Hooks-Catalog.md](../17-Hooks清单/Hooks清单-Hooks-Catalog.md)