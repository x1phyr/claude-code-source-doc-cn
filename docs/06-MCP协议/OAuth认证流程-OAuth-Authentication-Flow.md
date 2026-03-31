# OAuth 认证流程

## 概述

MCP 服务器支持 OAuth 2.0 认证，允许用户通过第三方身份提供商（如 Google、GitHub）授权访问。Claude Code 实现了完整的 OAuth 2.0 授权码流程，包括 PKCE 扩展。

## 核心文件

- [src/services/mcp/auth.ts](src/services/mcp/auth.ts) — OAuth 实现
- [src/services/mcp/oauthPort.ts](src/services/mcp/oauthPort.ts) — 端口管理
- [src/constants/oauth.ts](src/constants/oauth.ts) — OAuth 配置

## OAuth 流程

### 授权码流程 (Authorization Code Flow)

```
1. 发现授权服务器元数据
2. 注册客户端 (DCR)
3. 生成 PKCE 挑战
4. 打开浏览器授权
5. 回调接收授权码
6. 交换令牌
7. 存储令牌
```

### 发现流程

```typescript
async function fetchAuthServerMetadata(
  serverName: string,
  serverUrl: string,
  configuredMetadataUrl: string | undefined,
): Promise<AuthorizationServerMetadata> {
  // 优先使用配置的元数据 URL
  if (configuredMetadataUrl) {
    if (!configuredMetadataUrl.startsWith('https://')) {
      throw new Error('authServerMetadataUrl must use https://')
    }
    const response = await authFetch(configuredMetadataUrl)
    return OAuthMetadataSchema.parse(await response.json())
  }

  // RFC 9728 → RFC 8414 发现链
  try {
    const { authorizationServerMetadata } = await discoverOAuthServerInfo(
      serverUrl,
      { fetchFn }
    )
    if (authorizationServerMetadata) {
      return authorizationServerMetadata
    }
  } catch (err) {
    // 回退到传统路径感知探测
    logMCPDebug(serverName, `RFC 9728 discovery failed: ${errorMessage(err)}`)
  }

  // RFC 8414 直接探测
  // ...
}
```

### 客户端注册 (DCR)

```typescript
async function registerOAuthClient(
  metadata: AuthorizationServerMetadata,
  redirectUri: string,
): Promise<OAuthClientInformationFull> {
  const clientMetadata: OAuthClientMetadata = {
    client_name: 'Claude Code',
    redirect_uris: [redirectUri],
    grant_types: ['authorization_code', 'refresh_token'],
    response_types: ['code'],
    token_endpoint_auth_method: 'none', // PKCE
  }

  const response = await fetch(metadata.registration_endpoint, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(clientMetadata),
  })

  return OAuthClientInformationFull.parse(await response.json())
}
```

## PKCE 实现

### 生成挑战

```typescript
import { createHash, randomBytes } from 'crypto'

function generatePKCE(): { codeVerifier: string; codeChallenge: string } {
  // 生成随机验证器
  const codeVerifier = randomBytes(32).toString('base64url')

  // 计算挑战
  const codeChallenge = createHash('sha256')
    .update(codeVerifier)
    .digest('base64url')

  return { codeVerifier, codeChallenge }
}
```

### 授权请求

```typescript
const authUrl = new URL(metadata.authorization_endpoint)
authUrl.searchParams.set('response_type', 'code')
authUrl.searchParams.set('client_id', clientInfo.client_id)
authUrl.searchParams.set('redirect_uri', redirectUri)
authUrl.searchParams.set('code_challenge', codeChallenge)
authUrl.searchParams.set('code_challenge_method', 'S256')
authUrl.searchParams.set('state', state)
authUrl.searchParams.set('scope', scope)

// 打开浏览器
await openBrowser(authUrl.toString())
```

## 回调处理

### 本地服务器

```typescript
import { createServer, type Server } from 'http'

async function waitForCallback(port: number): Promise<CallbackResult> {
  return new Promise((resolve, reject) => {
    const server = createServer((req, res) => {
      const url = parse(req.url || '', true)

      // 检查 state 匹配
      if (url.query.state !== expectedState) {
        reject(new Error('State mismatch'))
        return
      }

      // 提取授权码
      const code = url.query.code
      if (code && typeof code === 'string') {
        resolve({ code, redirectUri })
      } else {
        reject(new Error('No authorization code'))
      }

      res.writeHead(200)
      res.end('Authorization successful! You can close this window.')
      server.close()
    })

    server.listen(port)
  })
}
```

### 端口管理

```typescript
// 查找可用端口
export async function findAvailablePort(startPort: number): Promise<number> {
  const port = startPort
  while (port < startPort + 100) {
    try {
      await new Promise((resolve, reject) => {
        const server = createServer()
        server.once('error', reject)
        server.once('listening', () => server.close(resolve))
        server.listen(port)
      })
      return port
    } catch {
      port++
    }
  }
  throw new Error('No available port found')
}

// 构建回调 URI
export function buildRedirectUri(port: number): string {
  return `http://localhost:${port}/callback`
}
```

## 令牌交换

### 获取令牌

```typescript
async function exchangeCodeForTokens(
  code: string,
  codeVerifier: string,
  clientInfo: OAuthClientInformation,
  metadata: AuthorizationServerMetadata,
  redirectUri: string,
): Promise<OAuthTokens> {
  const response = await fetch(metadata.token_endpoint, {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      grant_type: 'authorization_code',
      code,
      redirect_uri: redirectUri,
      client_id: clientInfo.client_id,
      code_verifier: codeVerifier,
    }),
  })

  if (!response.ok) {
    throw new OAuthError(await response.text())
  }

  return OAuthTokensSchema.parse(await response.json())
}
```

### 刷新令牌

```typescript
async function refreshTokens(
  refreshToken: string,
  clientInfo: OAuthClientInformation,
  metadata: AuthorizationServerMetadata,
): Promise<OAuthTokens> {
  const response = await fetch(metadata.token_endpoint, {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      grant_type: 'refresh_token',
      refresh_token: refreshToken,
      client_id: clientInfo.client_id,
    }),
  })

  if (!response.ok) {
    if (response.status === 400) {
      // invalid_grant - 需要重新授权
      throw new InvalidGrantError()
    }
    throw new OAuthError(await response.text())
  }

  return OAuthTokensSchema.parse(await response.json())
}
```

## 令牌存储

### 安全存储

```typescript
import { getSecureStorage } from '../../utils/secureStorage/index.js'

export class ClaudeAuthProvider implements OAuthClientProvider {
  private getStorageKey(serverUrl: string): string {
    return `mcp-oauth-${createHash('sha256').update(serverUrl).digest('hex')}`
  }

  async saveTokens(serverUrl: string, tokens: OAuthTokens): Promise<void> {
    const storage = getSecureStorage()
    const key = this.getStorageKey(serverUrl)
    await storage.set(key, {
      access_token: tokens.access_token,
      refresh_token: tokens.refresh_token,
      expires_at: tokens.expires_at,
    })
  }

  async getTokens(serverUrl: string): Promise<OAuthTokens | null> {
    const storage = getSecureStorage()
    const key = this.getStorageKey(serverUrl)
    return storage.get(key)
  }
}
```

### macOS Keychain

```typescript
// macOS 使用 Keychain 存储
export async function saveToKeychain(
  service: string,
  account: string,
  data: string,
): Promise<void> {
  // 使用 security 命令行工具
  await execFile('security', [
    'add-generic-password',
    '-a', account,
    '-s', service,
    '-w', data,
  ])
}
```

## 错误处理

### OAuth 错误类型

```typescript
import {
  InvalidGrantError,
  OAuthError,
  ServerError,
  TemporarilyUnavailableError,
  TooManyRequestsError,
} from '@modelcontextprotocol/sdk/server/auth/errors.js'

// 处理不同错误类型
function handleOAuthError(error: unknown): MCPOAuthFlowErrorReason {
  if (error instanceof InvalidGrantError) {
    return 'invalid_grant'
  }
  if (error instanceof TemporarilyUnavailableError) {
    return 'transient_retries_exhausted'
  }
  if (error instanceof TooManyRequestsError) {
    return 'transient_retries_exhausted'
  }
  return 'request_failed'
}
```

### 非标准错误码

```typescript
// Slack 等服务使用非标准错误码
const NONSTANDARD_INVALID_GRANT_ALIASES = new Set([
  'invalid_refresh_token',
  'expired_refresh_token',
  'token_expired',
])

// 标准化为 invalid_grant
export async function normalizeOAuthErrorBody(response: Response): Promise<Response> {
  if (!response.ok) return response

  const text = await response.text()
  let parsed
  try { parsed = JSON.parse(text) } catch { return new Response(text, response) }

  // 检查是否为 OAuth 错误响应
  if (OAuthTokensSchema.safeParse(parsed).success) {
    return new Response(text, response)
  }

  const result = OAuthErrorResponseSchema.safeParse(parsed)
  if (!result.success) return new Response(text, response)

  // 标准化非标准错误码
  const normalized = NONSTANDARD_INVALID_GRANT_ALIASES.has(result.data.error)
    ? { error: 'invalid_grant', error_description: result.data.error_description }
    : result.data

  return new Response(JSON.stringify(normalized), { status: 400 })
}
```

## 超时配置

```typescript
// OAuth 请求超时
const AUTH_REQUEST_TIMEOUT_MS = 30000

// 创建带超时的 fetch
function createAuthFetch(): FetchLike {
  return async (url, init) => {
    const timeoutSignal = AbortSignal.timeout(AUTH_REQUEST_TIMEOUT_MS)
    // 合并信号...
    return fetch(url, { ...init, signal: controller.signal })
  }
}
```

## 日志脱敏

```typescript
// 敏感 OAuth 参数
const SENSITIVE_OAUTH_PARAMS = [
  'state',
  'nonce',
  'code_challenge',
  'code_verifier',
  'code',
]

function redactSensitiveUrlParams(url: string): string {
  try {
    const parsedUrl = new URL(url)
    for (const param of SENSITIVE_OAUTH_PARAMS) {
      if (parsedUrl.searchParams.has(param)) {
        parsedUrl.searchParams.set(param, '[REDACTED]')
      }
    }
    return parsedUrl.toString()
  } catch {
    return url
  }
}
```

## 统计事件

```typescript
logEvent('tengu_mcp_oauth_flow_start', {
  serverName,
  transportType,
})

logEvent('tengu_mcp_oauth_flow_success', {
  serverName,
  durationMs,
})

logEvent('tengu_mcp_oauth_flow_error', {
  serverName,
  reason: errorReason,
  durationMs,
})

logEvent('tengu_mcp_oauth_refresh_failure', {
  serverName,
  reason: failureReason,
})
```

## 关键文件

- [src/services/mcp/auth.ts](src/services/mcp/auth.ts) — OAuth 实现
- [src/services/mcp/oauthPort.ts](src/services/mcp/oauthPort.ts) — 端口管理
- [src/constants/oauth.ts](src/constants/oauth.ts) — 配置常量
- [src/utils/secureStorage/](src/utils/secureStorage/) — 安全存储

## 关联文档

- [06-MCP协议/MCP协议实现-MCP-Protocol-Implementation.md](MCP协议实现-MCP-Protocol-Implementation.md)
- [06-MCP协议/MCP集成详解-MCP-Integration.md](MCP集成详解-MCP-Integration.md)