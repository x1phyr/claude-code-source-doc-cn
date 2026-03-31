# 沙箱实现

## 概述

沙箱是 Claude Code 的安全隔离机制，用于限制 Bash 命令的文件系统访问和网络访问，防止恶意或意外操作对系统造成损害。

## 核心文件

- [src/utils/sandbox/sandbox-adapter.ts](src/utils/sandbox/sandbox-adapter.ts) — 沙箱适配器
- [src/tools/BashTool/shouldUseSandbox.ts](src/tools/BashTool/shouldUseSandbox.ts) — 沙箱判断
- [src/components/sandbox/](src/components/sandbox/) — 沙箱 UI 组件

## 架构设计

### 适配器模式

`SandboxManager` 是对 `@anthropic-ai/sandbox-runtime` 的封装：

```typescript
export const SandboxManager: ISandboxManager = {
  // 自定义实现
  initialize,
  isSandboxingEnabled,
  isSandboxEnabledInSettings: getSandboxEnabledSetting,
  isPlatformInEnabledList,
  getSandboxUnavailableReason,
  isAutoAllowBashIfSandboxedEnabled,
  areUnsandboxedCommandsAllowed,
  isSandboxRequired,
  areSandboxSettingsLockedByPolicy,
  setSandboxSettings,
  getExcludedCommands,
  wrapWithSandbox,
  refreshConfig,
  reset,
  checkDependencies,

  // 转发到基础沙箱管理器
  getFsReadConfig: BaseSandboxManager.getFsReadConfig,
  getFsWriteConfig: BaseSandboxManager.getFsWriteConfig,
  getNetworkRestrictionConfig: BaseSandboxManager.getNetworkRestrictionConfig,
  // ...
}
```

## 配置转换

### 设置转换为运行时配置

```typescript
export function convertToSandboxRuntimeConfig(
  settings: SettingsJson,
): SandboxRuntimeConfig {
  const permissions = settings.permissions || {}

  // 提取网络域名
  const allowedDomains: string[] = []
  const deniedDomains: string[] = []

  // 从 WebFetch 规则提取域名
  for (const ruleString of permissions.allow || []) {
    const rule = permissionRuleValueFromString(ruleString)
    if (rule.toolName === WEB_FETCH_TOOL_NAME && rule.ruleContent?.startsWith('domain:')) {
      allowedDomains.push(rule.ruleContent.substring('domain:'.length))
    }
  }

  // 提取文件系统路径
  const allowWrite: string[] = ['.', getClaudeTempDir()]
  const denyWrite: string[] = []
  const denyRead: string[] = []

  // 从 Edit 规则提取写路径
  // 从 Read 规则提取读路径
  // ...

  return {
    network: {
      allowedDomains,
      deniedDomains,
      allowUnixSockets,
      allowLocalBinding,
      httpProxyPort,
      socksProxyPort,
    },
    filesystem: {
      denyRead,
      allowRead,
      allowWrite,
      denyWrite,
    },
    ignoreViolations,
    enableWeakerNestedSandbox,
    enableWeakerNetworkIsolation,
    ripgrep: ripgrepConfig,
  }
}
```

## 路径解析

### 权限规则路径约定

```typescript
/**
 * Claude Code 特殊路径前缀：
 * - `//path` → 绝对路径（从文件系统根目录）
 * - `/path` → 相对于设置文件目录
 * - `~/path` → 用户主目录（传递给 sandbox-runtime）
 * - `./path` 或 `path` → 相对路径（传递给 sandbox-runtime）
 */
export function resolvePathPatternForSandbox(
  pattern: string,
  source: SettingSource,
): string {
  // 处理 // 前缀 - 绝对路径
  if (pattern.startsWith('//')) {
    return pattern.slice(1) // "//.aws/**" → "/.aws/**"
  }

  // 处理 / 前缀 - 相对于设置文件目录
  if (pattern.startsWith('/') && !pattern.startsWith('//')) {
    const root = getSettingsRootPathForSource(source)
    return resolve(root, pattern.slice(1))
  }

  return pattern
}
```

### 沙箱文件系统路径

```typescript
export function resolveSandboxFilesystemPath(
  pattern: string,
  source: SettingSource,
): string {
  // 兼容旧语法：//path → /path
  if (pattern.startsWith('//')) return pattern.slice(1)
  // 展开路径（包括 ~）
  return expandPath(pattern, getSettingsRootPathForSource(source))
}
```

## 安全防护

### 设置文件保护

```typescript
// 始终拒绝写入设置文件，防止沙箱逃逸
const settingsPaths = SETTING_SOURCES.map(source =>
  getSettingsFilePathForSource(source),
).filter((p): p is string => p !== undefined)
denyWrite.push(...settingsPaths)
denyWrite.push(getManagedSettingsDropInDir())
```

### Git 仓库保护

```typescript
// 防止攻击者伪造 bare repo 逃逸沙箱
// Git 的 is_git_directory() 将包含 HEAD + objects/ + refs/ 的目录视为 bare repo
const bareGitRepoFiles = ['HEAD', 'objects', 'refs', 'hooks', 'config']
for (const dir of [originalCwd, cwd]) {
  for (const gitFile of bareGitRepoFiles) {
    const p = resolve(dir, gitFile)
    try {
      statSync(p)
      denyWrite.push(p)  // 存在则拒绝写入
    } catch {
      bareGitRepoScrubPaths.push(p)  // 不存在则清理
    }
  }
}
```

### 技能目录保护

```typescript
// 阻止写入 .claude/skills 目录
// Skills 与 commands/agents 具有相同的权限级别
denyWrite.push(resolve(originalCwd, '.claude', 'skills'))
if (cwd !== originalCwd) {
  denyWrite.push(resolve(cwd, '.claude', 'skills'))
}
```

## 网络隔离

### 域名控制

```typescript
// 当 allowManagedDomainsOnly 启用时，只使用策略设置中的域名
if (shouldAllowManagedSandboxDomainsOnly()) {
  const policySettings = getSettingsForSource('policySettings')
  for (const domain of policySettings?.sandbox?.network?.allowedDomains || []) {
    allowedDomains.push(domain)
  }
}
```

### Unix Socket 控制

```typescript
network: {
  allowUnixSockets: settings.sandbox?.network?.allowUnixSockets,
  allowAllUnixSockets: settings.sandbox?.network?.allowAllUnixSockets,
  allowLocalBinding: settings.sandbox?.network?.allowLocalBinding,
}
```

## 初始化流程

```typescript
async function initialize(sandboxAskCallback?: SandboxAskCallback): Promise<void> {
  // 防止重复初始化
  if (initializationPromise) {
    return initializationPromise
  }

  // 检查是否启用
  if (!isSandboxingEnabled()) {
    return
  }

  initializationPromise = (async () => {
    try {
      // 检测 git worktree 主仓库路径
      if (worktreeMainRepoPath === undefined) {
        worktreeMainRepoPath = await detectWorktreeMainRepoPath(getCwdState())
      }

      // 构建运行时配置
      const settings = getSettings_DEPRECATED()
      const runtimeConfig = convertToSandboxRuntimeConfig(settings)

      // 初始化基础沙箱管理器
      await BaseSandboxManager.initialize(runtimeConfig, wrappedCallback)

      // 订阅设置变更
      settingsSubscriptionCleanup = settingsChangeDetector.subscribe(() => {
        const newConfig = convertToSandboxRuntimeConfig(getSettings_DEPRECATED())
        BaseSandboxManager.updateConfig(newConfig)
      })
    } catch (error) {
      initializationPromise = undefined
      logForDebugging(`Failed to initialize sandbox: ${errorMessage(error)}`)
    }
  })()

  return initializationPromise
}
```

## 命令包装

```typescript
async function wrapWithSandbox(
  command: string,
  binShell?: string,
  customConfig?: Partial<SandboxRuntimeConfig>,
  abortSignal?: AbortSignal,
): Promise<string> {
  if (isSandboxingEnabled()) {
    if (initializationPromise) {
      await initializationPromise
    } else {
      throw new Error('Sandbox failed to initialize.')
    }
  }

  return BaseSandboxManager.wrapWithSandbox(
    command,
    binShell,
    customConfig,
    abortSignal,
  )
}
```

## 平台支持

### 支持的平台

- macOS
- Linux
- WSL2（不支持 WSL1）

```typescript
const isSupportedPlatform = memoize((): boolean => {
  return BaseSandboxManager.isSupportedPlatform()
})
```

### 平台限制

```typescript
// 限制特定平台启用沙箱
function isPlatformInEnabledList(): boolean {
  const enabledPlatforms = settings?.sandbox?.enabledPlatforms
  if (enabledPlatforms === undefined) {
    return true  // 未设置则全部启用
  }
  if (enabledPlatforms.length === 0) {
    return false  // 空数组则全部禁用
  }
  return enabledPlatforms.includes(getPlatform())
}
```

## 依赖检查

```typescript
const checkDependencies = memoize((): SandboxDependencyCheck => {
  const { rgPath, rgArgs } = ripgrepCommand()
  return BaseSandboxManager.checkDependencies({
    command: rgPath,
    args: rgArgs,
  })
})
```

## 排除命令

```typescript
// 获取不应被沙箱化的命令
function getExcludedCommands(): string[] {
  const settings = getSettings_DEPRECATED()
  return settings?.sandbox?.excludedCommands ?? []
}

// 添加到排除列表
export function addToExcludedCommands(
  command: string,
  permissionUpdates?: Array<{...}>,
): string {
  const existingExcludedCommands =
    existingSettings?.sandbox?.excludedCommands || []

  if (!existingExcludedCommands.includes(commandPattern)) {
    updateSettingsForSource('localSettings', {
      sandbox: {
        ...existingSettings?.sandbox,
        excludedCommands: [...existingExcludedCommands, commandPattern],
      },
    })
  }

  return commandPattern
}
```

## 违规处理

### 违规存储

```typescript
getSandboxViolationStore: BaseSandboxManager.getSandboxViolationStore,
annotateStderrWithSandboxFailures: BaseSandboxManager.annotateStderrWithSandboxFailures,
```

### 命令后清理

```typescript
cleanupAfterCommand: (): void => {
  BaseSandboxManager.cleanupAfterCommand()
  scrubBareGitRepoFiles()  // 清理可能被植入的 bare repo 文件
},
```

## 配置选项

### settings.json 中的沙箱配置

```json
{
  "sandbox": {
    "enabled": true,
    "autoAllowBashIfSandboxed": true,
    "allowUnsandboxedCommands": true,
    "excludedCommands": ["npm run test"],
    "enabledPlatforms": ["macos"],
    "network": {
      "allowedDomains": ["api.example.com"],
      "allowUnixSockets": ["*"],
      "allowLocalBinding": true
    },
    "filesystem": {
      "allowWrite": ["/path/to/allowed"],
      "denyWrite": ["/path/to/denied"],
      "allowRead": ["/path/to/read"],
      "denyRead": ["/path/to/block"]
    },
    "ignoreViolations": ["network"],
    "failIfUnavailable": false
  }
}
```

## 关键文件

- [src/utils/sandbox/sandbox-adapter.ts](src/utils/sandbox/sandbox-adapter.ts) — 适配器
- [src/tools/BashTool/shouldUseSandbox.ts](src/tools/BashTool/shouldUseSandbox.ts) — 判断逻辑
- [src/components/sandbox/](src/components/sandbox/) — UI 组件
- [src/commands/sandbox-toggle/](src/commands/sandbox-toggle/) — 切换命令

## 关联文档

- [03-工具系统/权限检查细节-Permission-Check-Details.md](../03-工具系统/权限检查细节-Permission-Check-Details.md)
- [05-权限安全/权限系统详解-Permission-System.md](../05-权限安全/权限系统详解-Permission-System.md)