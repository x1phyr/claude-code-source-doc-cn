# 对话框组件

## 概述

对话框组件用于用户确认、选择和输入，是 Claude Code 与用户交互的重要界面元素。

## 核心文件

- [src/components/ConfirmDialog.tsx](src/components/ConfirmDialog.tsx) — 确认对话框
- [src/components/PermissionDialog.tsx](src/components/PermissionDialog.tsx) — 权限对话框
- [src/components/GlobalSearchDialog.tsx](src/components/GlobalSearchDialog.tsx) — 搜索对话框

## 对话框类型

### 确认对话框

```typescript
export function ConfirmDialog({
  message,
  onConfirm,
  onCancel,
  confirmLabel = 'Yes',
  cancelLabel = 'No',
}: ConfirmDialogProps): React.ReactElement {
  const [selected, setSelected] = useState<'confirm' | 'cancel'>('cancel')

  useInput((input, key) => {
    if (key.leftArrow || key.rightArrow) {
      setSelected(s => s === 'confirm' ? 'cancel' : 'confirm')
    }
    if (key.return) {
      selected === 'confirm' ? onConfirm() : onCancel()
    }
    if (key.escape) {
      onCancel()
    }
  })

  return (
    <Box flexDirection="column" borderStyle="round" borderColor="yellow">
      <Box paddingX={1}>
        <Text>{message}</Text>
      </Box>
      <Box paddingX={1} justifyContent="center">
        <Text
          bold={selected === 'cancel'}
          inverse={selected === 'cancel'}
        >
          [{cancelLabel}]
        </Text>
        <Text> </Text>
        <Text
          bold={selected === 'confirm'}
          inverse={selected === 'confirm'}
        >
          [{confirmLabel}]
        </Text>
      </Box>
    </Box>
  )
}
```

### 选择对话框

```typescript
export function SelectDialog<T>({
  title,
  options,
  onSelect,
  onCancel,
}: SelectDialogProps<T>): React.ReactElement {
  const [selectedIndex, setSelectedIndex] = useState(0)

  useInput((input, key) => {
    if (key.upArrow) {
      setSelectedIndex(i => Math.max(0, i - 1))
    }
    if (key.downArrow) {
      setSelectedIndex(i => Math.min(options.length - 1, i + 1))
    }
    if (key.return) {
      onSelect(options[selectedIndex])
    }
    if (key.escape) {
      onCancel()
    }
  })

  return (
    <Box flexDirection="column" borderStyle="round">
      <Box paddingX={1}>
        <Text bold>{title}</Text>
      </Box>
      {options.map((option, index) => (
        <Box key={index} paddingX={1}>
          <Text
            bold={selectedIndex === index}
            inverse={selectedIndex === index}
          >
            {selectedIndex === index ? '› ' : '  '}
            {option.label}
          </Text>
        </Box>
      ))}
    </Box>
  )
}
```

### 多选对话框

```typescript
export function MultiSelectDialog<T>({
  title,
  options,
  onConfirm,
  onCancel,
}: MultiSelectDialogProps<T>): React.ReactElement {
  const [selectedIndex, setSelectedIndex] = useState(0)
  const [selected, setSelected] = useState<Set<number>>(new Set())

  useInput((input, key) => {
    if (key.upArrow) {
      setSelectedIndex(i => Math.max(0, i - 1))
    }
    if (key.downArrow) {
      setSelectedIndex(i => Math.min(options.length - 1, i + 1))
    }
    if (key.return || input === ' ') {
      setSelected(prev => {
        const next = new Set(prev)
        if (next.has(selectedIndex)) {
          next.delete(selectedIndex)
        } else {
          next.add(selectedIndex)
        }
        return next
      })
    }
    if (input === '\r') {  // Enter
      onConfirm(options.filter((_, i) => selected.has(i)))
    }
    if (key.escape) {
      onCancel()
    }
  })

  return (
    <Box flexDirection="column" borderStyle="round">
      <Box paddingX={1}>
        <Text bold>{title}</Text>
        <Text dimColor> (Space to select, Enter to confirm)</Text>
      </Box>
      {options.map((option, index) => (
        <Box key={index} paddingX={1}>
          <Text
            bold={selectedIndex === index}
            inverse={selectedIndex === index}
          >
            {selected.has(index) ? '☑' : '☐'}
            {' '}
            {option.label}
          </Text>
        </Box>
      ))}
    </Box>
  )
}
```

## 权限对话框

### 文件权限对话框

```typescript
export function FilePermissionDialog({
  filePath,
  operation,
  onAllow,
  onDeny,
}: FilePermissionDialogProps): React.ReactElement {
  const [rememberChoice, setRememberChoice] = useState(false)

  return (
    <Box flexDirection="column" borderStyle="round" borderColor="yellow">
      <Box paddingX={1}>
        <Text bold color="yellow">Permission Required</Text>
      </Box>
      <Box paddingX={1}>
        <Text>
          Allow <Text bold>{operation}</Text> on:
        </Text>
      </Box>
      <Box paddingX={1}>
        <Text color="cyan">{filePath}</Text>
      </Box>
      <Box paddingX={1}>
        <Text dimColor>
          [r] Remember choice  [y] Yes  [n] No  [a] Always allow
        </Text>
      </Box>
    </Box>
  )
}
```

### MCP 服务器对话框

```typescript
export function MCPServerApprovalDialog({
  serverName,
  config,
  onApprove,
  onDeny,
}: MCPServerApprovalDialogProps): React.ReactElement {
  return (
    <Box flexDirection="column" borderStyle="round">
      <Box paddingX={1}>
        <Text bold>MCP Server Approval</Text>
      </Box>
      <Box paddingX={1}>
        <Text>New MCP server wants to connect:</Text>
      </Box>
      <Box paddingX={1} flexDirection="column">
        <Text bold>{serverName}</Text>
        <Text dimColor>Command: {config.command}</Text>
        {config.args && (
          <Text dimColor>Args: {config.args.join(' ')}</Text>
        )}
      </Box>
      <Box paddingX={1}>
        <Text dimColor>[a] Approve  [d] Deny  [v] View details</Text>
      </Box>
    </Box>
  )
}
```

## 搜索对话框

### 全局搜索

```typescript
export function GlobalSearchDialog({
  isOpen,
  onClose,
}: GlobalSearchDialogProps): React.ReactElement {
  const [query, setQuery] = useState('')
  const [results, setResults] = useState<SearchResult[]>([])
  const [selectedIndex, setSelectedIndex] = useState(0)

  // 防抖搜索
  const debouncedSearch = useMemo(
    () => debounce((q: string) => {
      setResults(searchFiles(q))
    }, 200),
    [],
  )

  useInput((input, key) => {
    if (key.escape) {
      onClose()
      return
    }

    if (key.upArrow) {
      setSelectedIndex(i => Math.max(0, i - 1))
      return
    }

    if (key.downArrow) {
      setSelectedIndex(i => Math.min(results.length - 1, i + 1))
      return
    }

    if (key.return && results[selectedIndex]) {
      // 打开文件
      openFile(results[selectedIndex].path)
      onClose()
      return
    }

    // 输入搜索词
    if (input && !key.ctrl && !key.meta) {
      setQuery(prev => prev + input)
      debouncedSearch(query + input)
    }

    if (key.backspace) {
      setQuery(prev => prev.slice(0, -1))
      debouncedSearch(query.slice(0, -1))
    }
  }, { isActive: isOpen })

  if (!isOpen) return null

  return (
    <Box flexDirection="column" borderStyle="double">
      <Box paddingX={1}>
        <Text dimColor>Search: </Text>
        <Text bold>{query}</Text>
        <Text dimColor>_</Text>
      </Box>
      {results.slice(0, 10).map((result, index) => (
        <Box key={result.path} paddingX={1}>
          <Text
            bold={selectedIndex === index}
            inverse={selectedIndex === index}
          >
            {result.filename}
          </Text>
          <Text dimColor> - {result.path}</Text>
        </Box>
      ))}
    </Box>
  )
}
```

### 历史搜索

```typescript
export function HistorySearchDialog({
  messages,
  onSelect,
  onClose,
}: HistorySearchDialogProps): React.ReactElement {
  const [query, setQuery] = useState('')
  const [matches, setMatches] = useState<number[]>([])

  useEffect(() => {
    if (query.length < 2) {
      setMatches([])
      return
    }

    const indices: number[] = []
    messages.forEach((msg, index) => {
      if (getContentText(msg).toLowerCase().includes(query.toLowerCase())) {
        indices.push(index)
      }
    })
    setMatches(indices)
  }, [query, messages])

  return (
    <Box flexDirection="column" borderStyle="round">
      <Box paddingX={1}>
        <Text dimColor>Search history: </Text>
        <Text bold>{query}</Text>
      </Box>
      <Box paddingX={1}>
        <Text dimColor>{matches.length} matches found</Text>
      </Box>
    </Box>
  )
}
```

## 输入对话框

### 文本输入

```typescript
export function InputDialog({
  title,
  placeholder,
  onSubmit,
  onCancel,
}: InputDialogProps): React.ReactElement {
  const [value, setValue] = useState('')

  useInput((input, key) => {
    if (key.escape) {
      onCancel()
      return
    }

    if (key.return) {
      onSubmit(value)
      return
    }

    if (key.backspace) {
      setValue(v => v.slice(0, -1))
      return
    }

    if (input && !key.ctrl && !key.meta) {
      setValue(v => v + input)
    }
  })

  return (
    <Box flexDirection="column" borderStyle="round">
      <Box paddingX={1}>
        <Text bold>{title}</Text>
      </Box>
      <Box paddingX={1}>
        {value ? (
          <Text>{value}</Text>
        ) : (
          <Text dimColor>{placeholder}</Text>
        )}
        <Text inverse> </Text>
      </Box>
    </Box>
  )
}
```

## 模态叠加

### 模态管理

```typescript
export function ModalOverlay({
  isOpen,
  children,
  onClose,
}: ModalOverlayProps): React.ReactElement | null {
  if (!isOpen) return null

  return (
    <Box
      position="absolute"
      top={0}
      left={0}
      right={0}
      bottom={0}
      flexDirection="column"
      justifyContent="center"
      alignItems="center"
    >
      <Box
        borderStyle="double"
        borderColor="white"
        padding={1}
      >
        {children}
      </Box>
    </Box>
  )
}
```

## 关键文件

- [src/components/ConfirmDialog.tsx](src/components/ConfirmDialog.tsx) — 确认对话框
- [src/components/GlobalSearchDialog.tsx](src/components/GlobalSearchDialog.tsx) — 全局搜索
- [src/components/HistorySearchDialog.tsx](src/components/HistorySearchDialog.tsx) — 历史搜索
- [src/components/permissions/](src/components/permissions/) — 权限对话框

## 关联文档

- [15-UI组件/核心组件-Core-Components.md](核心组件-Core-Components.md)
- [03-工具系统/权限检查细节-Permission-Check-Details.md](../03-工具系统/权限检查细节-Permission-Check-Details.md)