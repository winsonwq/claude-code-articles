---
title: Claude Code 源码解析 - 安全性
author: 百万
digest: 从权限模式到危险模式检测，深入理解 Claude Code 如何构建多层安全防护体系
---

想象你是一家公司的安全总监。现在有个新员工要来上班，这个员工能力很强，但也很虎——给他一把锤子，他能把你家拆了。

你怎么设计他的权限？

- 全封闭？——他什么都做不了，能力废了
- 全开放？——第一天就能把你服务器格式化了
- **按需授权**？——给他锤子可以，但告诉他在哪里钉钉子

Claude Code 的安全设计，就是**按需授权**的哲学。

## 一、权限模式：用户体验与安全的平衡

Claude Code 有五种权限模式，让用户决定「这个 AI 我能信任到什么程度」：

```typescript
// src/utils/permissions/PermissionMode.ts
const PERMISSION_MODE_CONFIG = {
  default: { title: 'Default', symbol: '' },
  plan: { title: 'Plan Mode', symbol: '⏸' },
  acceptEdits: { title: 'Accept edits', symbol: '⏵⏵' },
  bypassPermissions: { title: 'Bypass Permissions', symbol: '⏵⏵' },
  dontAsk: { title: "Don't Ask", symbol: '⏵⏵' },
}
```

### 1.1 五种模式详解

| 模式 | 行为 | 适用场景 |
|------|------|----------|
| **default** | 每次危险操作询问 | 初期使用，熟悉项目 |
| **acceptEdits** | 自动接受文件编辑 | 信任 AI 改代码 |
| **plan** | 只读，不执行任何操作 | 代码审查 |
| **bypassPermissions** | 跳过所有权限检查 | 调试/开发模式 |
| **dontAsk** | 不询问，直接执行 | 高度信任 |

### 1.2 自动模式：AI 辅助判断

Anthropic 内部还有一种 `auto` 模式，用 AI 决定是否需要询问：

```typescript
// 当 TRANSCRIPT_CLASSIFIER 功能开启时可用
...(feature('TRANSCRIPT_CLASSIFIER') ? {
  auto: { title: 'Auto mode', color: 'warning' }
} : {})
```

这个模式的好处：**减少不必要的弹窗，同时保证安全**。AI 会根据上下文判断这个操作是否「合理」，如果偏离任务太远，才会询问用户。

## 二、权限规则系统：精细化控制

### 2.1 规则结构

```typescript
// src/types/permissions.ts
type PermissionRule = {
  source: 'userSettings' | 'projectSettings' | 'cliArg' | 'command'
  ruleBehavior: 'allow' | 'deny' | 'ask'
  ruleValue: { toolName: string; ruleContent?: string }
}
```

每个规则说三件事：
- **谁定的** — source
- **怎么处置** — behavior
- **管什么** — toolName + 可选的 content

### 2.2 规则匹配顺序

Claude Code 的规则匹配有严格的优先级：

```
1. deny 规则（最高优先级）
2. 内部可编辑路径检查
3. 安全检查
4. 工作目录检查
5. allow 规则
```

**deny 永远优先**。如果你说「禁止删除 .git」，那即使你又说「允许所有操作」，删除 .git 依然会被阻止。

### 2.3 规则来源

| 来源 | 说明 |
|------|------|
| `cliArg` | 命令行参数 `--allowed-tools` |
| `command` | 会话中的 `/allow` 命令 |
| `userSettings` | 用户配置 `~/.claude/settings.json` |
| `projectSettings` | 项目配置 `.claude/settings.json`（需信任） |

```typescript
// 示例：允许使用 Read 和 Edit，拒绝 Bash rm
{
  rules: [
    { toolName: 'Read', behavior: 'allow' },
    { toolName: 'Edit', behavior: 'allow' },
    { toolName: 'Bash', ruleContent: 'rm', behavior: 'deny' },
  ]
}
```

## 三、危险模式检测：保护敏感操作

这是 Claude Code 最有意思的设计之一：**不只是根据工具名判断，而是分析命令内容**。

### 3.1 Bash 危险模式

```typescript
// src/utils/permissions/dangerousPatterns.ts
export const DANGEROUS_BASH_PATTERNS = [
  // 代码解释器
  'python', 'python3', 'node', 'deno', 'ruby', 'perl', 'php', 'lua',
  
  // 包运行器
  'npx', 'bunx', 'npm run', 'yarn run', 'pnpm run', 'bun run',
  
  // Shell
  'bash', 'sh', 'zsh', 'fish',
  
  // 危险命令
  'eval', 'exec', 'env', 'xargs', 'sudo',
]
```

为什么这些危险？

- `python`、`node` —— 可以执行任意代码
- `npx`、`npm run` —— 可以运行任意脚本
- `eval`、`exec` —— 直接执行字符串

### 3.2 自动模式下的规则剥离

当你切换到 `auto` 模式时，Claude Code 会**自动剥离危险的 allow 规则**：

```typescript
// permissionSetup.ts
function stripDangerousRules(rules) {
  return rules.filter(rule => {
    // 如果允许 node:*, 剥离它 —— node 可以做任何事
    if (rule.toolName === 'Bash' && rule.ruleContent?.includes('node')) {
      return false
    }
    // ...
  })
}
```

这防止「开了自动模式，AI 用 node -e 'rm -rf /' 把你硬盘清空」的悲剧。

### 3.3 Ant 额外危险模式

Anthropic 内部还有一些额外的危险模式：

```typescript
...(process.env.USER_TYPE === 'ant' ? [
  'coo',           // 集群代码启动器
  'gh', 'gh api',  // GitHub API
  'curl', 'wget',  // 网络下载
  'kubectl', 'aws', 'gcloud',  // 云资源操作
] : [])
```

这些都是**可能造成外部影响**的操作，不只是本地破坏。

## 四、路径验证：文件系统安全

Claude Code 对文件路径有一套完整的安全检查：

### 4.1 路径验证流程

```typescript
// pathValidation.ts
function validatePath(path: string): PathCheckResult {
  // 1. 检查路径遍历 (../etc/passwd)
  if (containsPathTraversal(path)) return deny('Path traversal')
  
  // 2. 检查 UNC 网络路径
  if (containsVulnerableUncPath(path)) return deny('UNC path')
  
  // 3. 检查 shell 扩展语法
  if (path.includes('$') || path.includes('%')) {
    return deny('Shell expansion in path')
  }
  
  // 4. 检查危险删除模式
  if (isDangerousRemovalPath(path)) return deny('Dangerous path')
  
  // 5. 检查权限规则
  return checkPermissionRules(path)
}
```

### 4.2 危险路径检测

这些路径**永远不允许删除**：

```typescript
function isDangerousRemovalPath(path: string): boolean {
  // 通配符：rm *
  if (path === '*' || path.endsWith('/*')) return true
  
  // 根目录
  if (path === '/') return true
  
  // 家目录
  if (path === homedir()) return true
  
  // Windows 驱动器
  if (/^[A-Za-z]:\$/.test(path)) return true
}
```

### 4.3 路径遍历防护

```
攻击：echo "malicious" > /app/../../../etc/passwd
防御：检测到 ..，拒绝写入
```

Claude Code 会：
- 拒绝包含 `..` 的路径
- 拒绝 UNC 路径（`\\server\share`）
- 拒绝 shell 扩展语法（`$VAR`, `%VAR%`）

## 五、安全决策原因：透明可审计

每次权限决策，都有一个清晰的**原因**：

```typescript
// src/types/permissions.ts
type PermissionDecisionReason =
  | { type: 'rule'; rule: PermissionRule }        // 命中某条规则
  | { type: 'mode'; mode: PermissionMode }         // 权限模式决定
  | { type: 'classifier'; classifier: string }    // AI 分类器判断
  | { type: 'safetyCheck'; reason: string }        // 安全检查失败
  | { type: 'workingDir' }                         // 工作目录内
  | { type: 'other'; reason: string }              // 其他原因
```

用户可以问：「为什么你拒绝了这个操作？」——Claude Code 能给出具体原因。

## 六、设计思想：纵深防御

### 6.1 多层检查

Claude Code 的安全不是靠一道防线，而是**多层叠加**：

```
用户规则（allow/deny）
    ↓
内部路径检查（.claude 目录等）
    ↓
安全检查（危险命令、危险路径）
    ↓
工作目录检查
    ↓
最终规则匹配
```

每一层都可能拦截问题。攻击者需要**突破所有层**才能造成破坏。

### 6.2 deny 优先

```typescript
// 规则匹配顺序
deny > allow

// 示例
允许所有 Bash 操作
但 deny Bash(rm:*)
结果：rm 命令被拒绝，其他 Bash 命令允许
```

### 6.3 最小权限原则

Claude Code 默认**什么都不允许**，只有明确授权的操作才能执行。这和 Unix 的「默认禁止」哲学一致。

### 6.4 体验与安全平衡

权限模式让用户选择信任级别：
- 新项目 → `default`（保守）
- 熟悉项目后 → `acceptEdits`（放手）
- 代码审查 → `plan`（只读）
- 调试开发 → `bypassPermissions`（完全信任）

## 总结

Claude Code 的安全设计告诉我们：**安全不是限制，而是有控制的信任**。

通过权限模式，用户决定 AI 的自由度；通过规则系统，精细控制每个操作；通过危险模式检测，防止「好心办坏事」；通过路径验证，堵住常见攻击手法。

多层防御 + 透明可审计 + 用户控制 —— 这是 AI 驾驭工程的安全范式。

---

## 源码索引

| 功能 | 文件 |
|------|------|
| 权限类型定义 | `src/types/permissions.ts` |
| 权限模式 | `src/utils/permissions/PermissionMode.ts` |
| 危险模式 | `src/utils/permissions/dangerousPatterns.ts` |
| 路径验证 | `src/utils/permissions/pathValidation.ts` |
| 权限处理 | `src/utils/permissions/permissions.ts` |
