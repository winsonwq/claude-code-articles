---
title: Claude Code 源码解析 - 工具调用管理
author: 百万
digest: 从工具定义到权限管理，深入理解 Claude Code 如何构建精细化的工具调用系统
---

你知道 Claude Code 为什么能安全地帮程序员工作吗？

因为它不是在「执行命令」，而是在**调用工具**。

每个工具都有明确定义的输入输出、权限范围和执行约束。Claude Code 的工具系统，让 AI 的每一步操作都是**可预测、可控制、可审计**的。

## 一、工具架构：43 个工具的协同

Claude Code 内置 **43 个工具**，分为几大类。就像一个完整的工具箱，有螺丝刀、锤子、钳子，每种工具做不同的事。

**文件操作工具**负责读取、写入、编辑文件。Read 工具读取文件内容，Write 工具创建新文件，Edit 工具修改现有文件。

**代码执行工具**负责运行命令。BashTool 可以执行任何 Shell 命令，这是最强大的工具，也是最危险的。

**任务管理工具**让你创建、追踪、更新任务。想象有一个待办清单，Agent 可以往里面加任务。

**Agent 工具**让 Agent 调用其他 Agent。这是实现多 Agent 协作的关键。

**搜索工具**帮你找文件和内容。Glob 用模式匹配找文件，Grep 用正则搜内容。

**网络工具**让 Agent 能访问互联网。WebSearch 搜索网页，WebFetch 获取页面内容。

```bash
# 文件操作
FileReadTool, FileWriteTool, FileEditTool
GlobTool, GrepTool

# 代码执行
BashTool, PowerShellTool, REPLTool

# 任务管理
TaskCreateTool, TaskListTool, TaskUpdateTool
TaskStopTool, TaskOutputTool

# Agent
AgentTool, SkillTool

# 搜索
WebSearchTool, WebFetchTool

# MCP
MCPTool, ListMcpResourcesTool
```

### 1.1 工具定义结构

从代码层面看，每个工具都是一个对象，有名字、描述、输入schema和处理函数。

**name** 是工具的唯一标识符，就像工牌号。LLM 通过这个名字决定调用哪个工具。

**description** 是给 AI 看的说明书。AI 不知道工具的实现细节，只知道这个描述。所以描述要写清楚工具能做什么、什么时候用它。

**inputSchema** 定义了输入参数的格式。LLM 必须按照这个格式传参数，否则工具调用会失败。

**handler** 是实际执行逻辑。当 AI 决定调用工具后，handler 真正执行操作并返回结果。

```typescript
// src/Tool.ts
type Tool = {
  name: string           // 工具唯一标识
  category: string      // 工具分类
  description: string    // 工具描述（给 LLM 看）
  inputSchema: object    // 输入参数定义
  handler: Function      // 实际执行逻辑
  
  // 权限相关
  permissionMode?: PermissionMode  // 需要的权限级别
  dangerous?: boolean             // 是否危险操作
  requiresConfirmation?: boolean   // 是否需要确认
}
```

这种设计的精妙之处在于：**工具是自描述的**。LLM 只看 description 就能决定要不要用这个工具，不需要知道实现细节。

## 二、工具分类：职责清晰的分工

Claude Code 的 43 个工具按职责分成几类。分类的目的不是限制，而是**让 AI 更容易选择**。

### 2.1 文件操作工具

文件操作是最常用的工具。Read、Write、Edit 是三剑客。

**FileReadTool** 读取文件内容。AI 需要了解代码或配置时用它。这个工具是只读的，不会修改任何东西。

**FileWriteTool** 创建新文件。如果文件已存在，会报错。AI 应该用它来创建新文件，而不是覆盖已有文件。

**FileEditTool** 修改现有文件。它使用精确的文本替换来修改文件内容，比 sed 更安全，因为可以预览替换结果。

```typescript
// 文件操作工具的危险级别
// Read - 低危险，只读不修改
// Write - 中危险，创建文件可能覆盖重要内容
// Edit - 中危险，修改可能引入错误
```

### 2.2 代码执行工具

代码执行工具是最危险的类别。BashTool 可以执行**任何 Shell 命令**，包括删除文件、格式化硬盘。

**PowerShellTool** 在 Windows 上执行 PowerShell 命令。

**REPLTool** 启动交互式解释器（Python、Node 等），AI 可以在里面执行代码片段。

```typescript
// 代码执行工具是"万能钥匙"
// 有了 BashTool，AI 理论上可以做任何事
// 所以必须有严格的权限控制
```

### 2.3 任务管理工具

任务管理工具让 AI 能够追踪多步骤工作。

当你让 AI 开发一个功能时，这个功能可能需要很多步骤：阅读代码、理解需求、编写代码、写测试、提交 PR......任务工具让 AI 能够把这个大任务分解成小任务，一步步完成。

```typescript
// 任务创建
TaskCreateTool: {
  name: string,           // 任务名称
  description?: string,    // 任务描述
  status?: 'pending' | 'in_progress' | 'completed'
}

// 任务更新
TaskUpdateTool: {
  taskId: string,
  status?: string,
  completionMessage?: string
}
```

## 三、工具调用流程：从 LLM 到执行

当 LLM 决定调用工具时，Claude Code 经历一系列检查和执行步骤。

**第一步：解析**。LLM 输出工具名和参数，系统解析成结构化数据。

**第二步：权限检查**。系统检查当前权限模式下，这个工具是否允许调用。

**第三步：路径验证**（文件操作）。如果是文件操作，系统检查目标路径是否安全。

**第四步：执行**。通过权限检查后，真正执行工具的 handler。

**第五步：结果转换**。工具返回的结果被转换成 LLM 能理解的格式。

**第六步：返回**。结果返回给 LLM，LLM 决定下一步。

```
LLM 决定调用工具
       ↓
  解析工具名和参数
       ↓
  权限检查（是否允许调用）
       ↓
  路径验证（文件操作检查路径）
       ↓
  执行工具 handler
       ↓
  结果转换（转为 LLM 可理解的格式）
       ↓
  返回给 LLM 继续
```

### 3.1 权限检查

权限检查是工具调用的守门人。它根据权限模式和工具定义，决定是否允许这次调用。

首先检查工具是否在允许列表。如果不在，直接拒绝。

然后检查权限模式。不同模式下，同一个工具的行为不同。比如 `plan` 模式下，所有修改操作都被禁止。

最后检查工具的 `requiresConfirmation` 属性。高危工具需要用户确认。

```typescript
// 权限检查流程
function checkToolPermission(tool: Tool, input: object): PermissionResult {
  // 1. 检查工具是否在允许列表
  if (!context.allowedTools.includes(tool.name)) {
    return { behavior: 'deny', reason: 'Tool not allowed' }
  }
  
  // 2. 检查权限模式
  if (tool.requiresConfirmation) {
    return { behavior: 'ask', message: 'Confirm?' }
  }
  
  // 3. 危险操作额外检查
  if (tool.dangerous && context.mode === 'default') {
    return { behavior: 'ask' }
  }
  
  return { behavior: 'allow' }
}
```

### 3.2 路径验证

文件操作工具在执行前会验证路径。这是为了防止路径遍历攻击。

想象一下，AI 被诱导执行 `read(/path/to/some/file)`，但实际读取了 `/etc/passwd`。这就是路径遍历攻击。

Claude Code 的路径验证会：
- 检查是否有 `..` 遍历
- 检查是否是危险路径（`/etc`、`~/.ssh`）
- 检查是否匹配权限规则

```typescript
// 检查项
checkPathValidation(path) {
  // 1. 路径遍历检查 (../etc/passwd)
  if (containsPathTraversal(path)) return DENY
  
  // 2. 危险路径检查 (/etc, ~/.ssh)
  if (isDangerousPath(path)) return DENY
  
  // 3. 权限规则检查
  if (!matchesAllowRule(path)) return DENY
  
  return ALLOW
}
```

## 四、工具编排：复杂任务的分解

Claude Code 不是每次只调用一个工具，它可以**并行调用多个工具**，也可以**链式调用**形成工具链。

### 4.1 工具调用限制

为了防止 AI 进入无限循环，Claude Code 有调用次数限制。

每个回合最多调用 5 次工具。这是防止 AI 在一个回合里做太多事。

总调用次数有限制。当超过 100 次时，强制停止。这逼迫 AI 在有限步骤内完成任务。

```typescript
// 防止无限循环
const MAX_TOOL_CALLS_PER_TURN = 5
const MAX_TOTAL_TOOL_CALLS = 100

// 超过限制强制停止
if (totalCalls > MAX_TOTAL_TOOL_CALLS) {
  throw new Error('Too many tool calls')
}
```

### 4.2 并行工具调用

Claude Code 支持 LLM 同时调用多个工具。这叫做**并行调用**。

比如 AI 需要读取三个文件来理解上下文，它可以同时发起三个 Read 调用，而不是串行等待。

```typescript
// LLM 可以请求并行调用
{
  "tool_calls": [
    { "name": "Read", "input": { "file": "a.txt" } },
    { "name": "Read", "input": { "file": "b.txt" } },
    { "name": "Glob", "input": { "pattern": "*.ts" } }
  ]
}
```

并行调用让 AI 效率更高。但也要注意：不是所有工具都可以并行，有依赖关系的工具必须串行。

## 五、BashTool：最危险的工具

BashTool 是 Claude Code 里最强大的工具，也是最危险的。它可以执行**任意 Shell 命令**。

### 5.1 命令分类

不是所有命令都一样危险。Claude Code 把命令分成几类：

**安全命令**：只读操作，如 `ls`、`cat`、`git status`。这些命令不会修改系统状态。

**中等风险命令**：`npm install`、`git add` 等。这些会修改项目文件，但影响有限。

**危险命令**：`rm -rf`、`dd`、`mkfs` 等。这些可能造成永久性破坏。

**灾难级命令**：`curl | bash`、`wget -O- | sh`。下载远程脚本并执行，这是最危险的。

```typescript
// 命令分类
type CommandClassification = {
  safe: ['ls', 'cat', 'git status', 'git log'],
  moderate: ['npm install', 'pip install', 'git add'],
  dangerous: ['rm -rf', 'dd', 'mkfs'],
  critical: ['curl | bash', 'wget -O- | sh']
}
```

### 5.2 安全措施

Claude Code 为 BashTool 设计了多层安全措施：

**命令白名单**。默认只允许安全命令。危险命令需要用户明确授权。

**危险模式检测**。系统会检测危险命令模式，如 `rm -rf`、`dd` 等。

**Shell 扩展阻止**。`$VAR`、`$(cmd)` 等会被阻止，防止命令注入。

**执行超时**。长时间运行的命令会被强制终止。

```typescript
// 安全措施
// 1. 命令白名单
const ALLOWED_COMMANDS = ['ls', 'cat', 'git', ...]

// 2. 危险模式检测
if (isDangerousPattern(command)) {
  return { behavior: 'ask', dangerLevel: 'high' }
}

// 3. Shell 扩展阻止
if (containsShellExpansion(command)) {
  return { behavior: 'deny' }
}
```

### 5.3 交互式确认

对于高危命令，Claude Code 会弹出确认框，让用户决定是否执行。

确认框会显示命令的危险等级和可能的影响。如果用户不确定，可以选择拒绝。

```typescript
// 高危命令需要用户确认
if (isDangerousCommand(command)) {
  return {
    behavior: 'ask',
    message: `This command will:\n${explainDanger(command)}\n\nProceed?`,
    metadata: {
      dangerLevel: 'high',
      canUndo: isUndoable(command)
    }
  }
}
```

## 六、工具与权限模式

Claude Code 有五种权限模式，工具的可用性取决于当前模式。

**default 模式**是标准模式，每次危险操作都会询问。

**acceptEdits 模式**自动允许编辑操作，但危险命令仍需确认。

**plan 模式**只读模式，所有修改操作都被禁止。

**bypassPermissions 模式**跳过所有权限检查，危险！

**dontAsk 模式**不询问，直接执行所有操作。

```typescript
// 权限模式对工具的影响
const PERMISSION_MODE_EFFECTS = {
  'default': {
    BashTool: { behavior: 'ask', dangerous: true },
    FileEditTool: { behavior: 'ask' },
  },
  'acceptEdits': {
    BashTool: { behavior: 'ask', dangerous: true },
    FileEditTool: { behavior: 'allow' },
  },
  'plan': {
    FileReadTool: { behavior: 'allow' },
    BashTool: { behavior: 'deny' },
  },
  'bypassPermissions': {
    '*': { behavior: 'allow' }
  }
}
```

## 七、MCP 工具：扩展系统

MCP（Model Context Protocol）让 Claude Code 能够调用**外部工具**。

MCP 服务器可以是任何提供工具的服务。比如文件系统服务器、Git 服务器、数据库服务器等。

当 Claude Code 连接到一个 MCP 服务器时，服务器上的工具会自动出现在 Claude Code 的工具列表里。

```typescript
// MCP 工具调用流程
async function callMcpTool(server: string, tool: string, input: object) {
  // 1. 获取 MCP 服务器连接
  const client = mcpClients.get(server)
  
  // 2. 调用远程工具
  const result = await client.callTool(tool, input)
  
  // 3. 结果转换
  return transformMcpResult(result)
}
```

MCP 的强大之处在于：**任何系统都可以通过 MCP 暴露自己的工具**。Claude Code 本身不需要知道怎么连接数据库，只要数据库提供了 MCP 服务器，就能用自然语言查询数据库。

## 八、设计思想：工具即边界

Claude Code 的工具系统体现了几个核心设计思想。

**最小权限原则**。每个工具只暴露必要的功能，不暴露危险操作。比如 BashTool 不会直接暴露 `rm -rf`，需要特殊权限。

**可审计性**。每次工具调用都有完整日志，包括：调用时间、工具名和参数、执行结果、权限决策。这些日志对于排查问题和安全审计至关重要。

**可扩展性**。新工具只需实现 Tool 接口即可加入系统。这让 Claude Code 的功能可以不断扩展。

## 总结

Claude Code 的工具系统是 **AI 驾驭工程的核心体现**：

- 43 个工具，职责清晰
- 权限检查贯穿始终
- 路径验证防止攻击
- BashTool 最危险也最受保护
- MCP 扩展无限可能

工具是 AI 能力的边界。边界越清晰，控制越精确。

---

## 源码索引

| 功能 | 文件 |
|------|------|
| 工具定义 | `src/Tool.ts` |
| 工具列表 | `src/constants/tools.ts` |
| Bash 权限分类 | `src/utils/permissions/bashClassifier.ts` |
| 路径验证 | `src/utils/permissions/pathValidation.ts` |
| MCP 工具 | `src/services/mcp/client.ts` |
