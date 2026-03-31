# 危险命令检测

## 概述

危险命令检测是 Claude Code 的安全防护层，识别并阻止潜在的恶意或破坏性 Bash 命令，防止对系统和数据造成损害。

## 核心文件

- [src/utils/permissions/dangerousPatterns.ts](src/utils/permissions/dangerousPatterns.ts) — 危险模式
- [src/tools/BashTool/bashPermissions.ts](src/tools/BashTool/bashPermissions.ts) — 权限检查
- [src/tools/PowerShellTool/gitSafety.ts](src/tools/PowerShellTool/gitSafety.ts) — Git 安全

## 检测机制

### 危险模式匹配

```typescript
// dangerousPatterns.ts
export const DANGEROUS_PATTERNS: DangerousPattern[] = [
  // 文件系统破坏
  { pattern: /^rm\s+-rf\s+\/$/, severity: 'critical', message: '拒绝删除根目录' },
  { pattern: /^rm\s+-rf\s+~$/, severity: 'critical', message: '拒绝删除用户目录' },
  { pattern: /^rm\s+-rf\s+\.\//, severity: 'high', message: '删除当前目录需确认' },

  // 磁盘操作
  { pattern: /^mkfs/, severity: 'critical', message: '拒绝格式化磁盘' },
  { pattern: /^dd\s+if=/, severity: 'high', message: 'dd 命令需要确认' },

  // 权限提升
  { pattern: /^sudo\s+/, severity: 'medium', message: 'sudo 命令需要确认' },
  { pattern: /^su\s+/, severity: 'medium', message: '切换用户需要确认' },
  { pattern: /^chmod\s+777/, severity: 'medium', message: '完全开放权限需确认' },

  // 网络操作
  { pattern: /^curl.*\|\s*bash/, severity: 'high', message: '远程脚本执行有风险' },
  { pattern: /^wget.*\|\s*bash/, severity: 'high', message: '远程脚本执行有风险' },

  // 进程管理
  { pattern: /^kill\s+-9\s+1$/, severity: 'critical', message: '拒绝杀死 init 进程' },
  { pattern: /^pkill\s+-9/, severity: 'high', message: '强制杀死进程需确认' },
]

export interface DangerousPattern {
  pattern: RegExp
  severity: 'critical' | 'high' | 'medium' | 'low'
  message: string
}
```

### 检测函数

```typescript
export function detectDangerousCommand(command: string): DangerousPattern | null {
  // 标准化命令
  const normalizedCommand = normalizeCommand(command)

  // 检查每个模式
  for (const dangerous of DANGEROUS_PATTERNS) {
    if (dangerous.pattern.test(normalizedCommand)) {
      return dangerous
    }
  }

  // 检查组合风险
  return detectCombinedRisks(normalizedCommand)
}

function normalizeCommand(command: string): string {
  // 移除多余空格
  let normalized = command.trim().replace(/\s+/g, ' ')

  // 处理引号
  normalized = normalizeQuotes(normalized)

  // 展开环境变量（部分）
  normalized = expandCommonEnvVars(normalized)

  return normalized
}
```

## 沙箱阻止命令

```typescript
// shouldUseSandbox.ts
export const SANDBOX_BLOCKED_COMMANDS = [
  'rm -rf /',
  'mkfs',
  'dd if=',
  ':(){ :|:& };:',  // Fork bomb
  'chmod -R 777 /',
  'chown -R',
]

export function isCommandBlockedForSandbox(command: string): boolean {
  const normalized = command.trim()

  for (const blocked of SANDBOX_BLOCKED_COMMANDS) {
    if (normalized.includes(blocked)) {
      return true
    }
  }

  return false
}

export const SANDBOX_PROTECTED_PATHS = [
  '/',
  '/etc',
  '/usr',
  '/bin',
  '/sbin',
  '/lib',
  '/lib64',
  '/boot',
  '/dev',
  '/proc',
  '/sys',
]

export function isProtectedPath(path: string): boolean {
  const resolved = resolve(path)

  for (const protected_path of SANDBOX_PROTECTED_PATHS) {
    if (resolved === protected_path || resolved.startsWith(protected_path + '/')) {
      return true
    }
  }

  return false
}
```

## Bash 权限检查

### 命令解析

```typescript
// bashPermissions.ts
export async function checkBashPermissions(
  command: string,
  context: ToolUseContext,
  canUseTool: CanUseToolFn,
): Promise<PermissionResult> {
  // 1. 检查危险命令
  const dangerous = detectDangerousCommand(command)
  if (dangerous) {
    if (dangerous.severity === 'critical') {
      return {
        behavior: 'deny',
        message: dangerous.message,
      }
    }
    return {
      behavior: 'ask',
      message: dangerous.message,
      decisionReason: { type: 'safetyCheck', reason: dangerous.message },
    }
  }

  // 2. 解析子命令
  const subcommands = parseSubcommands(command)

  // 3. 检查每个子命令
  const results = new Map<string, PermissionResult>()
  for (const subcmd of subcommands) {
    const result = await checkSingleCommand(subcmd, context, canUseTool)
    results.set(subcmd, result)
  }

  // 4. 汇总结果
  return aggregateResults(results)
}
```

### 子命令解析

```typescript
import { parse } from 'shell-quote'

export function parseSubcommands(command: string): string[] {
  const tokens = parse(command)
  const subcommands: string[] = []
  let current: string[] = []

  for (const token of tokens) {
    if (token === '&&' || token === '||' || token === ';') {
      if (current.length > 0) {
        subcommands.push(current.join(' '))
        current = []
      }
    } else if (typeof token === 'string') {
      current.push(token)
    }
  }

  if (current.length > 0) {
    subcommands.push(current.join(' '))
  }

  return subcommands
}
```

### 管道命令检查

```typescript
export function checkPipedCommands(
  command: string,
  context: ToolUseContext,
): PermissionResult {
  const pipeSegments = command.split('|').map(s => s.trim())

  // 检查每个管道段
  for (const segment of pipeSegments) {
    const dangerous = detectDangerousCommand(segment)
    if (dangerous?.severity === 'critical') {
      return {
        behavior: 'deny',
        message: `管道中包含危险命令: ${dangerous.message}`,
      }
    }
  }

  // 检查远程脚本下载执行
  const remoteScriptPattern = /(curl|wget|fetch)\s+.*\|\s*(bash|sh|zsh)/
  if (remoteScriptPattern.test(command)) {
    return {
      behavior: 'ask',
      message: '执行远程下载的脚本存在安全风险，请确认',
      decisionReason: { type: 'safetyCheck', reason: 'Remote script execution' },
    }
  }

  return { behavior: 'allow' }
}
```

## Git 安全检查

```typescript
// gitSafety.ts
export const DANGEROUS_GIT_COMMANDS = [
  'git push --force',
  'git push -f',
  'git reset --hard',
  'git clean -fdx',
  'git checkout --',
  'git rebase -i',
  'git filter-branch',
]

export function checkGitCommandSafety(
  command: string,
  context: ToolUseContext,
): PermissionResult {
  // 检查危险 Git 命令
  for (const dangerous of DANGEROUS_GIT_COMMANDS) {
    if (command.includes(dangerous)) {
      return {
        behavior: 'ask',
        message: `Git 命令 '${dangerous}' 可能导致数据丢失，请确认`,
        decisionReason: { type: 'safetyCheck', reason: `Dangerous git command: ${dangerous}` },
      }
    }
  }

  // 检查 force push 到主分支
  const forcePushPattern = /git\s+push.*(--force|-f).*main|master/
  if (forcePushPattern.test(command)) {
    return {
      behavior: 'ask',
      message: '强制推送到主分支可能影响其他开发者，请确认',
      decisionReason: { type: 'safetyCheck', reason: 'Force push to main branch' },
    }
  }

  return { behavior: 'allow' }
}
```

## 敏感路径保护

```typescript
// filesystem.ts
export const SENSITIVE_PATHS = [
  // 系统配置
  '/etc/passwd',
  '/etc/shadow',
  '/etc/sudoers',
  // SSH 密钥
  '~/.ssh/id_rsa',
  '~/.ssh/id_ed25519',
  // 凭证文件
  '~/.netrc',
  '~/.pgpass',
  '~/.my.cnf',
  // 云服务凭证
  '~/.aws/credentials',
  '~/.gcp/keyfile.json',
  '~/.config/gcloud',
]

export function checkSensitivePathAccess(
  path: string,
  operation: 'read' | 'write',
): PermissionResult {
  const expandedPath = expandPath(path)

  for (const sensitive of SENSITIVE_PATHS) {
    const expandedSensitive = expandPath(sensitive)

    if (expandedPath === expandedSensitive) {
      if (operation === 'write') {
        return {
          behavior: 'deny',
          message: `拒绝修改敏感文件: ${path}`,
        }
      }
      return {
        behavior: 'ask',
        message: `访问敏感文件需要确认: ${path}`,
        decisionReason: { type: 'safetyCheck', reason: 'Sensitive file access' },
      }
    }
  }

  return { behavior: 'allow' }
}
```

## 组合风险检测

```typescript
function detectCombinedRisks(command: string): DangerousPattern | null {
  // 检测重定向风险
  if (command.includes('> /dev/') || command.includes('> /proc/')) {
    return {
      pattern: /./,
      severity: 'high',
      message: '写入设备文件可能损坏系统',
    }
  }

  // 检测后台执行的危险命令
  if (command.includes('&') && isPotentiallyDestructive(command)) {
    return {
      pattern: /./,
      severity: 'medium',
      message: '后台执行的破坏性命令需要确认',
    }
  }

  // 检测环境变量注入
  if (/[A-Z_]+=\$\(.*\)/.test(command)) {
    return {
      pattern: /./,
      severity: 'medium',
      message: '命令替换赋值可能有风险',
    }
  }

  return null
}

function isPotentiallyDestructive(command: string): boolean {
  const destructiveKeywords = ['rm', 'delete', 'truncate', 'drop', 'format']
  return destructiveKeywords.some(kw => command.includes(kw))
}
```

## 安全建议

### 生成安全替代

```typescript
export function generateSafeAlternative(
  command: string,
  danger: DangerousPattern,
): string | null {
  if (/^rm\s+-rf/.test(command)) {
    // 建议使用安全删除
    return command.replace('rm -rf', 'rm -ri')
  }

  if (/^chmod\s+777/.test(command)) {
    // 建议最小权限
    return command.replace('chmod 777', 'chmod 755')
  }

  if (/curl.*\|\s*bash/.test(command)) {
    // 建议先下载检查
    return `# 先下载检查脚本内容:\n# curl -fsSL <url> | less\n# 确认安全后再执行`
  }

  return null
}
```

## 关键文件

- [src/utils/permissions/dangerousPatterns.ts](src/utils/permissions/dangerousPatterns.ts) — 危险模式
- [src/tools/BashTool/bashPermissions.ts](src/tools/BashTool/bashPermissions.ts) — Bash 权限
- [src/tools/BashTool/readOnlyValidation.ts](src/tools/BashTool/readOnlyValidation.ts) — 只读检查
- [src/tools/PowerShellTool/gitSafety.ts](src/tools/PowerShellTool/gitSafety.ts) — Git 安全
- [src/utils/permissions/filesystem.ts](src/utils/permissions/filesystem.ts) — 文件系统保护

## 关联文档

- [03-工具系统/沙箱实现-Sandbox-Implementation.md](沙箱实现-Sandbox-Implementation.md)
- [03-工具系统/权限检查细节-Permission-Check-Details.md](权限检查细节-Permission-Check-Details.md)
- [05-权限安全/权限系统详解-Permission-System.md](../05-权限安全/权限系统详解-Permission-System.md)