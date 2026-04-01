---
title: Claude Code 源码解析 - Agent 架构
author: 百万
digest: 从 Agent 定义到内置 Agent 实现，深入理解 Claude Code 如何构建多Agent协作系统
---

![头图](/home/aqiu/.openclaw/workspace/wechat-articles/drafts/images/t04-header.png)


想象你要盖一栋房子。你不可能自己一个人搬砖、和泥、设计结构——你需要不同工种的工人：木工、电工、水泥工...

Claude Code 的 Agent 架构就是**给 AI 配了一支施工队**：有专门探索代码库的 Explore Agent，有专门做计划的 Plan Agent，还有什么都能干的通用 Agent。

## 一、Agent 定义：Markdown 即配置

![Agent定义](/home/aqiu/.openclaw/workspace/wechat-articles/drafts/images/t04-section1.png)

Claude Code 的 Agent 定义简单到令人发指——**用 Markdown 写 Agent 配置**。

```markdown
---
name: MyAgent
description: 当你需要做某件事时使用这个 Agent
model: haiku
maxTurns: 10
---

你是一个专门做 XXX 的 Agent...
```

### 1.1 Agent 定义结构

```typescript
// loadAgentsDir.ts
type BaseAgentDefinition = {
  agentType: string           // Agent 唯一标识
  whenToUse: string           // 何时使用这个 Agent
  tools?: string[]            // 允许使用的工具
  disallowedTools?: string[]  // 禁止使用的工具
  model?: string              // 模型选择
  effort?: EffortValue        // 努力程度
  permissionMode?: PermissionMode  // 权限模式
  maxTurns?: number           // 最大回合数
  skills?: string[]           // 预加载的技能
  memory?: 'user' | 'project' | 'local'  // 记忆范围
  background?: boolean        // 是否后台运行
}
```

### 1.2 三种 Agent 来源

| 来源 | 说明 |
|------|------|
| `built-in` | 内置 Agent（Explore、Plan 等） |
| `plugin` | 插件提供的 Agent |
| `userSettings` / `projectSettings` | 用户/项目自定义 Agent |

Claude Code 会合并所有来源，同名的 Agent 后加载的覆盖先加载的。

## 二、内置 Agent：开箱即用的专业队

![内置Agent](/home/aqiu/.openclaw/workspace/wechat-articles/drafts/images/t04-section2.png)

### 2.1 Explore Agent — 快速搜索专家

```typescript
// built-in/exploreAgent.ts
export const EXPLORE_AGENT = {
  agentType: 'Explore',
  whenToUse: 'Fast agent specialized for exploring codebases...',
  disallowedTools: ['Agent', 'Edit', 'Write', ...],
  model: 'haiku',  // 速度优先
  omitClaudeMd: true,  // 不需要 CLAUDE.md
  getSystemPrompt: () => getExploreSystemPrompt(),
}
```

**设计亮点**：
- **只读模式** — 不能修改任何文件，只能搜索
- **速度优先** — 用 haiku 模型，快速返回
- **省略 CLAUDE.md** — 探索不需要项目规范
- **禁止工具** — Edit、Write、Agent tool 全部禁用

### 2.2 Plan Agent — 计划制定者

Plan Agent 专门用于制定计划，和 `plan` 权限模式配合使用。

### 2.3 General Purpose Agent — 万能工人

默认的通用 Agent，能做任何事。

## 三、Agent 运行机制：Fork 模式

当主 Agent 需要调用子 Agent 时，Claude Code 使用 **Fork 模式**：

```typescript
// runAgent.ts
async function runAgent(agentDefinition: AgentDefinition) {
  // 1. 继承主 Agent 的上下文
  const context = createSubagentContext(parentContext)
  
  // 2. 设置独立的工作目录（可选）
  if (agentDefinition.isolation === 'worktree') {
    context.cwd = createIsolatedWorktree()
  }
  
  // 3. 独立的工具权限
  context.tools = resolveAgentTools(agentDefinition)
  
  // 4. 运行子 Agent
  return await query(context)
}
```

### 3.1 Fork 的优势

1. **隔离性** — 子 Agent 的操作不影响主 Agent 状态
2. **并行性** — 可以同时运行多个子 Agent
3. **资源控制** — 独立的 token 预算和 maxTurns

### 3.2 上下文传递

```
主 Agent 上下文
├── System Prompt（部分传递）
├── 工具列表（按 Agent 定义过滤）
├── 记忆（按 memory scope 决定）
└── MCP Servers（可继承可添加）
```

## 四、工具解析：Agent 能用什么

```typescript
// agentToolUtils.ts
function resolveAgentTools(agent: AgentDefinition): Tools {
  // 1. 从主工具列表开始
  let tools = getMainTools()
  
  // 2. 应用 allowedTools 过滤器
  if (agent.tools) {
    tools = tools.filter(t => agent.tools.includes(t.name))
  }
  
  // 3. 应用 disallowedTools 过滤器
  if (agent.disallowedTools) {
    tools = tools.filter(t => !agent.disallowedTools.includes(t.name))
  }
  
  return tools
}
```

工具解析规则：
- **默认**：使用所有工具
- **`tools`**：白名单，只允许列表中的工具
- **`disallowedTools`**：黑名单，禁止列表中的工具
- 两者同时存在：**先 allow 再 deny**

## 五、记忆隔离：Agent 的独立记忆空间
![记忆隔离](/home/aqiu/.openclaw/workspace/wechat-articles/drafts/images/t04-section3.png)
```typescript
// agentMemory.ts
type AgentMemoryScope = 'user' | 'project' | 'local'
// Agent 可以有自己的记忆空间
const agentMemory = {
  'user': '~/.claude/agents/<agentType>/memory/',
  'project': '<project>/.claude/agents/<agentType>/memory/',
  'local': '<session>/memory/'
}
```
**local 记忆**是 Claude Code 的创新：每个 Agent 运行可以有独立的临时记忆，不污染全局记忆空间。
## 六、Agent 协作模式

### 6.1 主从模式

```
用户 → 主 Agent → 子 Agent (Explore/Plan)
                ↓
           收集结果
                ↓
           主 Agent 汇总
```

主 Agent 负责任务分解、结果汇总；子 Agent 负责执行特定子任务。

### 6.2 并行模式

```
主 Agent → [Agent A] ← 同时运行
         → [Agent B] ← 
         → [Agent C] ←
```

多个子 Agent 并行工作，主 Agent 等待所有结果后继续。

### 6.3 验证模式

```
主 Agent → 完成任务
         → 触发验证 Agent
         → 验证结果
         → 通过/失败
```

专门的验证 Agent（VERIFICATION_AGENT）检查主 Agent 的输出。

## 七、设计思想：专业化与隔离

### 7.1 专业化原则

每个 Agent 只做一件事：
- Explore → 搜索
- Plan → 计划
- Verification → 验证

专业化的 Agent 比通用 Agent 更容易优化和维护。

### 7.2 隔离原则

- **工具隔离** — Explore 不能写文件
- **记忆隔离** — local 记忆不污染全局
- **工作目录隔离** — worktree 模式独立 git 状态
### 7.3 组合原则

简单积木可以组合出复杂系统：
- 主 Agent + Explore = 带代码探索能力的助手
- 主 Agent + Plan = 带计划能力的助手
- 主 Agent + Explore + Plan = 全能助手

## 总结

Claude Code 的 Agent 架构告诉我们：**专业化 + 隔离 = 可维护的复杂系统**。

用 Markdown 定义 Agent 让任何人都能创建自定义 Agent；Fork 模式让子 Agent 不影响主 Agent；工具过滤和记忆隔离让每个 Agent 各司其职。
如果你在设计多 Agent 系统，问自己：每个 Agent 的边界在哪里？如何保证隔离又支持协作？
---
## 源码索引

| 功能 | 文件 |
|------|------|
| Agent 定义加载 | `src/tools/AgentTool/loadAgentsDir.ts` |
| Agent 运行 | `src/tools/AgentTool/runAgent.ts` |
| 内置 Agent | `src/tools/AgentTool/builtInAgents.ts` |
| Explore Agent | `src/tools/AgentTool/built-in/exploreAgent.ts` |
| Plan Agent | `src/tools/AgentTool/built-in/planAgent.ts` |
| 工具解析 | `src/tools/AgentTool/agentToolUtils.ts` |
| 记忆管理 | `src/tools/AgentTool/agentMemory.ts` |
