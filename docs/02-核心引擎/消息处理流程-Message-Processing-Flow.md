# 消息处理流程

## 概述

消息处理是 Claude Code 会话的核心流程，负责处理用户输入、工具调用和助手响应的整个生命周期。

## 核心文件

- [src/QueryEngine.ts](src/QueryEngine.ts) — 查询引擎
- [src/types/message.ts](src/types/message.ts) — 消息类型
- [src/utils/messages.ts](src/utils/messages.ts) — 消息工具

## 消息类型

### 类型定义

```typescript
type Message =
  | UserMessage
  | AssistantMessage
  | ProgressMessage
  | AttachmentMessage
  | SystemMessage
  | StreamEvent
  | TombstoneMessage
  | ToolUseSummaryMessage

type UserMessage = {
  role: 'user'
  content: string | ContentBlockParam[]
  id: string
  timestamp: number
  type?: 'user'
  isMeta?: boolean
  isCompactSummary?: boolean
}

type AssistantMessage = {
  role: 'assistant'
  content: ContentBlockParam[]
  id: string
  timestamp: number
  toolUse?: ToolUseBlock[]
  isApiErrorMessage?: boolean
}

type SystemMessage = {
  role: 'system'
  type: 'system'
  content: string
  id: string
  timestamp: number
}

type ProgressMessage = {
  type: 'progress'
  id: string
  uuid: UUID
  parentUuid?: UUID
  timestamp: number
  // ...
}
```

## 查询引擎

### 核心循环

```typescript
export class QueryEngine {
  async *query(userMessage: UserMessage): AsyncGenerator<Message, void> {
    // 1. 添加用户消息到上下文
    this.context.messages.push(userMessage)

    // 2. 检查是否需要压缩
    if (await this.shouldCompact()) {
      yield* this.performCompaction()
    }

    // 3. 构建请求
    const request = await this.buildRequest()

    // 4. 流式调用 API
    const stream = queryModelWithStreaming(request)

    // 5. 处理流式响应
    for await (const event of stream) {
      if (event.type === 'stream_event') {
        yield this.processStreamEvent(event.event)
      } else if (event.type === 'assistant') {
        // 6. 处理助手消息
        const assistantMessage = event
        this.context.messages.push(assistantMessage)

        // 7. 处理工具调用
        if (assistantMessage.toolUse) {
          yield* this.processToolCalls(assistantMessage.toolUse)
        }
      }
    }
  }
}
```

### 消息规范化

```typescript
export function normalizeMessagesForAPI(
  messages: Message[],
  tools: Tool[],
): MessageParam[] {
  return messages
    .filter(m => m.type !== 'progress')  // 过滤进度消息
    .map(m => {
      if (m.type === 'user') {
        return {
          role: 'user',
          content: normalizeUserContent(m.message.content),
        }
      }
      if (m.type === 'assistant') {
        return {
          role: 'assistant',
          content: normalizeAssistantContent(m.message.content),
        }
      }
      // ...
    })
}
```

## 工具调用处理

### 工具调用流程

```typescript
async *processToolCalls(
  toolCalls: ToolUseBlock[],
): AsyncGenerator<Message, void> {
  for (const toolCall of toolCalls) {
    // 1. 查找工具
    const tool = this.tools.find(t => t.name === toolCall.name)
    if (!tool) {
      yield this.createToolNotFoundError(toolCall)
      continue
    }

    // 2. 检查权限
    const permission = await checkPermissions(tool, toolCall.input, this.context)
    if (permission.behavior === 'deny') {
      yield this.createPermissionDeniedMessage(toolCall, permission.message)
      continue
    }
    if (permission.behavior === 'ask') {
      yield* this.requestPermission(toolCall, permission.message)
      continue
    }

    // 3. 执行工具
    yield this.createToolStartMessage(toolCall)

    try {
      const result = await tool.call(
        toolCall.input,
        this.context,
        this.canUseTool,
        this.parentMessage,
        (progress) => this.emitProgress(toolCall, progress)
      )

      // 4. 处理结果
      yield this.createToolResultMessage(toolCall, result)
    } catch (error) {
      yield this.createToolErrorMessage(toolCall, error)
    }
  }

  // 5. 如果有工具调用，继续查询
  if (toolCalls.length > 0) {
    yield* this.continueQuery()
  }
}
```

### 工具结果处理

```typescript
function createToolResultMessage(
  toolCall: ToolUseBlock,
  result: ToolResult,
): UserMessage {
  return {
    role: 'user',
    content: [{
      type: 'tool_result',
      tool_use_id: toolCall.id,
      content: formatToolResult(result.output),
    }],
    id: generateId(),
    timestamp: Date.now(),
  }
}

function formatToolResult(output: unknown): string | ContentBlockParam[] {
  if (typeof output === 'string') {
    // 截断过长输出
    if (output.length > MAX_TOOL_RESULT_LENGTH) {
      return output.slice(0, MAX_TOOL_RESULT_LENGTH) + '\n...[truncated]'
    }
    return output
  }
  // 处理结构化输出
  return JSON.stringify(output, null, 2)
}
```

## 流式事件

### 事件类型

```typescript
type StreamEvent =
  | ContentBlockStartEvent
  | ContentBlockDeltaEvent
  | ContentBlockStopEvent
  | MessageStartEvent
  | MessageDeltaEvent
  | MessageStopEvent

type ContentBlockStartEvent = {
  type: 'content_block_start'
  index: number
  content_block: TextBlock | ToolUseBlock
}

type ContentBlockDeltaEvent = {
  type: 'content_block_delta'
  index: number
  delta: TextDelta | InputJsonDelta
}

type ContentBlockStopEvent = {
  type: 'content_block_stop'
  index: number
}
```

### 事件处理

```typescript
function processStreamEvent(event: StreamEvent): Message | null {
  switch (event.type) {
    case 'content_block_start':
      if (event.content_block.type === 'text') {
        return {
          type: 'assistant',
          content: [{ type: 'text', text: '' }],
          id: generateId(),
          timestamp: Date.now(),
        }
      }
      // 工具调用开始
      return {
        type: 'tool_use_start',
        toolName: event.content_block.name,
        toolId: event.content_block.id,
      }

    case 'content_block_delta':
      if (event.delta.type === 'text_delta') {
        return {
          type: 'text_delta',
          delta: event.delta.text,
        }
      }
      // JSON 输入增量
      return {
        type: 'input_delta',
        delta: event.delta.partial_json,
      }

    case 'content_block_stop':
      return { type: 'content_block_stop', index: event.index }
  }
}
```

## 附件消息

### 附件类型

```typescript
type AttachmentType =
  | 'file_content'
  | 'plan_file_reference'
  | 'plan_mode'
  | 'invoked_skills'
  | 'task_status'
  | 'skill_discovery'
  | 'tool_delta'
  | 'agent_listing'
  | 'mcp_instructions'

type AttachmentMessage = {
  type: 'attachment'
  attachment: {
    type: AttachmentType
    // 类型特定字段
  }
  id: string
  uuid: UUID
  timestamp: number
}
```

### 附件注入

```typescript
function getAttachmentMessages(context: ToolUseContext): AttachmentMessage[] {
  const attachments: AttachmentMessage[] = []

  // 文件内容附件
  for (const [path, content] of context.readFileState) {
    attachments.push(createAttachmentMessage({
      type: 'file_content',
      path,
      content,
    }))
  }

  // 技能附件
  const skillAttachment = createSkillAttachmentIfNeeded(context.agentId)
  if (skillAttachment) {
    attachments.push(skillAttachment)
  }

  // 工具增量附件
  for (const att of getDeferredToolsDeltaAttachment(context.options.tools, ...)) {
    attachments.push(createAttachmentMessage(att))
  }

  return attachments
}
```

## 系统消息

### 压缩边界

```typescript
type SystemCompactBoundaryMessage = SystemMessage & {
  type: 'system'
  subtype: 'compact_boundary'
  compactMetadata: {
    trigger: 'auto' | 'manual'
    preCompactTokenCount: number
    timestamp: number
    preservedSegment?: {
      headUuid: UUID
      anchorUuid: UUID
      tailUuid: UUID
    }
  }
}

function createCompactBoundaryMessage(
  trigger: 'auto' | 'manual',
  preCompactTokenCount: number,
  lastMessageUuid?: UUID,
): SystemCompactBoundaryMessage {
  return {
    role: 'system',
    type: 'system',
    subtype: 'compact_boundary',
    content: '',
    id: generateId(),
    uuid: generateUUID(),
    timestamp: Date.now(),
    compactMetadata: {
      trigger,
      preCompactTokenCount,
      timestamp: Date.now(),
    },
  }
}
```

## 消息存储

### 会话持久化

```typescript
export async function saveSession(
  messages: Message[],
  sessionId: string,
): Promise<void> {
  const sessionPath = getTranscriptPath(sessionId)
  const lines = messages.map(m => JSON.stringify(m)).join('\n')
  await writeFile(sessionPath, lines)
}

export async function loadSession(sessionId: string): Promise<Message[]> {
  const sessionPath = getTranscriptPath(sessionId)
  const content = await readFile(sessionPath, 'utf-8')
  return content.trim().split('\n').map(line => JSON.parse(line))
}
```

### 消息索引

```typescript
// 构建消息索引以支持快速查找
function buildMessageIndex(messages: Message[]): Map<UUID, Message> {
  const index = new Map()
  for (const message of messages) {
    index.set(message.uuid, message)
  }
  return index
}

// 查找消息
function findMessage(messages: Message[], uuid: UUID): Message | undefined {
  return messages.find(m => m.uuid === uuid)
}

// 查找最后一条助手消息
function getLastAssistantMessage(messages: Message[]): AssistantMessage | undefined {
  for (let i = messages.length - 1; i >= 0; i--) {
    if (messages[i].type === 'assistant') {
      return messages[i] as AssistantMessage
    }
  }
  return undefined
}
```

## 错误处理

### API 错误消息

```typescript
function createSystemAPIErrorMessage(
  error: APIError,
  delayMs: number,
  attempt: number,
  maxRetries: number,
): SystemMessage {
  return {
    role: 'system',
    type: 'system',
    subtype: 'api_retry',
    content: `API error (${error.status}). Retrying in ${Math.round(delayMs / 1000)}s (attempt ${attempt}/${maxRetries})...`,
    id: generateId(),
    timestamp: Date.now(),
  }
}
```

### 工具错误

```typescript
function createToolErrorMessage(
  toolCall: ToolUseBlock,
  error: unknown,
): UserMessage {
  const errorMessage = error instanceof Error
    ? error.message
    : String(error)

  return {
    role: 'user',
    content: [{
      type: 'tool_result',
      tool_use_id: toolCall.id,
      content: `Error: ${errorMessage}`,
      is_error: true,
    }],
    id: generateId(),
    timestamp: Date.now(),
  }
}
```

## 关键文件

- [src/QueryEngine.ts](src/QueryEngine.ts) — 查询引擎
- [src/types/message.ts](src/types/message.ts) — 消息类型
- [src/utils/messages.ts](src/utils/messages.ts) — 消息工具
- [src/utils/sessionStorage.ts](src/utils/sessionStorage.ts) — 会话存储

## 关联文档

- [02-核心引擎/查询引擎-Query-Engine.md](查询引擎-Query-Engine.md)
- [02-核心引擎/状态管理-State-Management.md](状态管理-State-Management.md)
- [08-上下文管理/上下文压缩算法-Context-Compression-Algorithms.md](../08-上下文管理/上下文压缩算法-Context-Compression-Algorithms.md)