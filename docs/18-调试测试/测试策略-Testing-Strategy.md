# 测试策略

## 概述

Claude Code 使用多种测试方法确保代码质量和功能正确性，包括单元测试、集成测试和端到端测试。

## 核心原则

1. **测试关键路径**：专注于用户最常用的功能
2. **隔离测试**：每个测试独立运行，不依赖其他测试
3. **模拟外部依赖**：避免真实 API 调用和文件系统操作
4. **快速反馈**：测试应该在几秒内完成

## 测试框架

### Bun Test

Claude Code 使用 Bun 内置测试框架：

```typescript
import { test, expect, describe, beforeEach, afterEach } from 'bun:test'

describe('MyModule', () => {
  beforeEach(() => {
    // 测试前准备
  })

  afterEach(() => {
    // 测试后清理
  })

  test('should work correctly', () => {
    const result = myFunction('input')
    expect(result).toBe('expected')
  })

  test('should handle errors', () => {
    expect(() => throwError()).toThrow()
  })
})
```

### 异步测试

```typescript
test('async operation', async () => {
  const result = await asyncFunction()
  expect(result).toBeDefined()
})

test('with timeout', async () => {
  const result = await slowOperation()
  expect(result).toBe('done')
}, { timeout: 5000 })  // 5 秒超时
```

## 单元测试

### 工具函数测试

```typescript
import { test, expect, describe } from 'bun:test'
import { errorMessage, isAbortError } from '../errors'

describe('errorMessage', () => {
  test('extracts message from Error', () => {
    const error = new Error('test error')
    expect(errorMessage(error)).toBe('test error')
  })

  test('converts non-Error to string', () => {
    expect(errorMessage('string error')).toBe('string error')
    expect(errorMessage(123)).toBe('123')
    expect(errorMessage(null)).toBe('null')
  })
})

describe('isAbortError', () => {
  test('identifies AbortError', () => {
    const error = new AbortError('aborted')
    expect(isAbortError(error)).toBe(true)
  })

  test('identifies DOMException AbortError', () => {
    const error = new DOMException('aborted', 'AbortError')
    expect(isAbortError(error)).toBe(true)
  })

  test('returns false for other errors', () => {
    const error = new Error('other')
    expect(isAbortError(error)).toBe(false)
  })
})
```

### 解析器测试

```typescript
describe('parseSlashCommand', () => {
  test('parses simple command', () => {
    const result = parseSlashCommand('/help')
    expect(result).toEqual({
      commandName: 'help',
      args: '',
    })
  })

  test('parses command with args', () => {
    const result = parseSlashCommand('/model opus')
    expect(result).toEqual({
      commandName: 'model',
      args: 'opus',
    })
  })

  test('returns null for non-slash input', () => {
    const result = parseSlashCommand('help')
    expect(result).toBeNull()
  })
})
```

## 集成测试

### 文件系统测试

```typescript
import { test, expect, beforeEach, afterEach } from 'bun:test'
import { mkdir, writeFile, rm } from 'fs/promises'
import { join } from 'path'

describe('File operations', () => {
  const testDir = join(process.cwd(), '.test-temp')

  beforeEach(async () => {
    await mkdir(testDir, { recursive: true })
  })

  afterEach(async () => {
    await rm(testDir, { recursive: true, force: true })
  })

  test('reads file correctly', async () => {
    const filePath = join(testDir, 'test.txt')
    await writeFile(filePath, 'test content')

    const content = await readFile(filePath)
    expect(content).toBe('test content')
  })
})
```

### API 测试

```typescript
import { test, expect, mock } from 'bun:test'

describe('API client', () => {
  test('handles successful response', async () => {
    // Mock fetch
    const originalFetch = global.fetch
    global.fetch = mock(() =>
      Promise.resolve(new Response(JSON.stringify({ data: 'test' })))
    )

    try {
      const result = await apiClient.get('/endpoint')
      expect(result.data).toBe('test')
    } finally {
      global.fetch = originalFetch
    }
  })

  test('handles error response', async () => {
    global.fetch = mock(() =>
      Promise.resolve(new Response(null, { status: 500 }))
    )

    await expect(apiClient.get('/endpoint')).rejects.toThrow()
  })
})
```

## Mock 和 Stub

### 函数 Mock

```typescript
import { mock } from 'bun:test'

test('calls callback', () => {
  const callback = mock()
  doSomething(callback)
  expect(callback).toHaveBeenCalled()
  expect(callback).toHaveBeenCalledWith('arg1', 'arg2')
})
```

### 模块 Mock

```typescript
import { mock, expect } from 'bun:test'

// Mock 整个模块
mock.module('../external', () => ({
  fetchData: () => Promise.resolve({ data: 'mocked' }),
}))

const { fetchData } = await import('../external')
const result = await fetchData()
expect(result.data).toBe('mocked')
```

### 时间 Mock

```typescript
import { mock, afterEach } from 'bun:test'

describe('time-dependent code', () => {
  afterEach(() => {
    mock.restore()
  })

  test('uses mocked time', () => {
    const now = new Date('2024-01-01')
    mock.date(now)

    const result = getTimestamp()
    expect(result).toBe(1704067200000)
  })
})
```

## 测试状态管理

### 重置单例

```typescript
// 用于测试的重置函数
export function _resetForTesting(): void {
  singleton = null
  initializationState = 'not-started'
}

// 在测试中使用
afterEach(() => {
  MySingleton._resetForTesting()
})
```

### 测试上下文

```typescript
function createTestContext(): ToolUseContext {
  return {
    getAppState: () => ({
      messages: [],
      isProcessing: false,
      totalTokensUsed: 0,
    }),
    readFileState: new Map(),
    addNotification: mock(),
    setStreamMode: mock(),
  }
}
```

## 快照测试

```typescript
import { test, expect } from 'bun:test'

test('output matches snapshot', () => {
  const output = formatMessage({
    type: 'user',
    content: 'Hello',
  })

  expect(output).toMatchSnapshot()
})

test('updates snapshot', () => {
  const output = formatMessage({ /* ... */ })
  expect(output).toMatchSnapshot('custom name')
})
```

## 测试覆盖率

### 运行覆盖率

```bash
# 运行测试并生成覆盖率报告
bun test --coverage

# 指定覆盖率阈值
bun test --coverage --coverage-reporter=text --coverage-threshold=80
```

### 覆盖率配置

```json
{
  "coverage": {
    "include": ["src/**/*.ts"],
    "exclude": ["src/**/*.test.ts", "src/types/**"],
    "reporter": ["text", "html"],
    "threshold": {
      "lines": 80,
      "functions": 75,
      "branches": 70
    }
  }
}
```

## 测试组织

### 目录结构

```
src/
├── utils/
│   ├── errors.ts
│   ├── errors.test.ts    # 测试文件与源文件同目录
│   ├── parser.ts
│   └── parser.test.ts
└── services/
    ├── api/
    │   ├── client.ts
    │   └── client.test.ts
    └── lsp/
        ├── manager.ts
        └── manager.test.ts
```

### 测试命名

```typescript
// 测试文件：module.test.ts
// 测试描述：should + 预期行为

describe('moduleName', () => {
  describe('functionName', () => {
    test('should return expected value', () => {})
    test('should handle error case', () => {})
    test('should validate input', () => {})
  })
})
```

## 测试最佳实践

### 1. 测试行为而非实现

```typescript
// ❌ 不好：测试实现细节
test('internal state changes', () => {
  const obj = new MyClass()
  obj.process()
  expect(obj._internalState).toBe('processed')
})

// ✅ 好：测试公共行为
test('returns processed result', () => {
  const obj = new MyClass()
  const result = obj.process()
  expect(result).toBe('processed')
})
```

### 2. 使用描述性的测试名称

```typescript
// ❌ 不好
test('works', () => {})

// ✅ 好
test('returns empty array when no items found', () => {})
```

### 3. 一个测试一个断言

```typescript
// ❌ 不好：多个断言
test('user creation', () => {
  const user = createUser('John')
  expect(user.name).toBe('John')
  expect(user.id).toBeDefined()
  expect(user.createdAt).toBeInstanceOf(Date)
})

// ✅ 好：分开测试
test('user creation sets name', () => {
  const user = createUser('John')
  expect(user.name).toBe('John')
})

test('user creation generates id', () => {
  const user = createUser('John')
  expect(user.id).toBeDefined()
})
```

### 4. 清理副作用

```typescript
describe('file operations', () => {
  let tempFiles: string[] = []

  afterEach(async () => {
    for (const file of tempFiles) {
      await unlink(file).catch(() => {})
    }
    tempFiles = []
  })

  test('creates file', async () => {
    const path = await createFile('content')
    tempFiles.push(path)
    expect(await exists(path)).toBe(true)
  })
})
```

## 关键文件

- 各模块的 `*.test.ts` 文件
- [src/test-utils/](src/test-utils/) — 测试工具函数

## 关联文档

- [18-调试测试/调试工具-Debugging-Tools.md](调试工具-Debugging-Tools.md)
- [19-参考手册/类型参考-Types-Reference.md](../19-参考手册/类型参考-Types-Reference.md)