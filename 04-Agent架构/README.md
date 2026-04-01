---
title: Claude Code 源码解析 - Agent 架构
author: 百万
digest: 从 Agent 定义到内置 Agent 实现，深入理解 Claude Code 如何构建多Agent协作系统
---

想象你要盖一栋房子。你不可能自己一个人搬砖、和泥、设计结构——你需要不同工种的工人：木工、电工、水泥工...

Claude Code 的 Agent 架构就是**给 AI 配了一支施工队**：有专门探索代码库的 Explore Agent，有专门做计划的 Plan Agent，还有什么都能干的通用 Agent。

## 一、Agent 定义：Markdown 即配置

Claude Code 的设计者做了一个非常聪明的决定：**用 Markdown 写 Agent 配置**。这意味着任何人都可以创建自定义 Agent，不需要懂代码，只需要会写 Markdown。

为什么用 Markdown？因为 Markdown 本身就是一种配置语言。它有 frontmatter（`---`之间的部分），可以放元数据；正文可以放提示词。天然适合定义 Agent。

```markdown
---
name: MyAgent
description: 当你需要做某件事时使用这个 Agent
model: haiku
maxTurns: 10
---

你是一个专门做 XXX 的 Agent...
```

这就是一个完整的 Agent 定义。`name` 是标识符，`description` 告诉主 Agent 什么时候该调用它，`model` 指定用什么模型，`maxTurns` 限制它能思考多少轮。

### 1.1 Agent 定义结构

从代码层面看，Agent 的定义结构非常清晰。Claude Code 使用 TypeScript 类型系统来约束 Agent 的所有属性。

`agentType` 是 Agent 的唯一标识符，就像工牌号。`whenToUse` 是描述文本，告诉系统什么时候应该召唤这个 Agent。`tools` 和 `disallowedTools` 控制这个 Agent 能用什么工具、不能用什么工具。`model` 指定使用哪个 AI 模型——可以用快速的 haiku，也可以用强大的 opus。`memory` 控制 Agent 的记忆范围，是私有还是团队共享。

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

这种设计的精妙之处在于：**所有属性都是可选的**。你只需要定义你关心的属性，其他都走默认值。这让创建 Agent 变得非常简单。

### 1.2 三种 Agent 来源

Claude Code 的 Agent 有三种来源，就像员工有三种招聘渠道：

**内置 Agent（built-in）** 是 Claude Code 官方提供的，包括 Explore、Plan 等。这些 Agent 经过精心调优，开箱即用。

**插件 Agent（plugin）** 是第三方开发者提供的，通过插件系统集成进来。

**自定义 Agent** 是用户自己创建的，存放在用户配置（`~/.claude/`）或项目配置（`.claude/`）中。

```typescript
// Claude Code 会合并所有来源，同名的 Agent 后加载的覆盖先加载的
// 这意味着你可以用自己的 Agent 覆盖内置 Agent 的行为
```

## 二、内置 Agent：开箱即用的专业队

Claude Code 自带几个专业 Agent，相当于配备了一支精干的专家团队。

### 2.1 Explore Agent — 快速搜索专家

Explore Agent 是专门做代码探索的 Agent。它的核心设计理念是：**只读、迅速、不打扰**。

当你需要在一整个代码库里找一个函数在哪里、某个 API 被谁调用、或者了解一个模块的结构时，就用 Explore Agent。它不会修改任何文件，只会搜索和阅读。

它使用 haiku 模型——这是最快最便宜的模型，因为探索任务不需要深度推理，只需要快速扫描。

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

注意 `disallowedTools`——Explore Agent 被禁止使用 Edit、Write、Agent 等工具。这是一种**强制约束**，从根源上防止 Explore 误操作文件。

`omitClaudeMd: true` 是另一个精妙的设计。Explore 不需要读取项目的 CLAUDE.md 规范，因为它只做搜索，不做实现。这节省了大量上下文空间。

### 2.2 Plan Agent — 计划制定者

Plan Agent 专门用于制定计划。当你需要规划一个复杂任务的执行步骤时，可以用它先制定一个详细的计划，然后交给主 Agent 执行。

Plan Agent 和 `plan` 权限模式配合使用——在 Plan 模式下，所有修改操作都被禁止，只能制定计划。

### 2.3 General Purpose Agent — 万能工人

默认的主 Agent，什么都能干。它调用子 Agent 完成特定任务，自己负责任务分解和结果汇总。

## 三、Agent 运行机制：Fork 模式

当主 Agent 需要调用子 Agent 时，Claude Code 使用 **Fork 模式**。这就像主工程师把任务分配给实习生——主上下文复制一份给子 Agent，但子 Agent 的操作不会影响主 Agent 的状态。

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

Fork 模式有三个关键步骤：

**第一步：复制上下文**。主 Agent 的系统提示词、工具列表、记忆等都被复制一份。这保证子 Agent 知道自己在做什么任务。

**第二步：隔离工作目录**（可选）。如果 Agent 声明了 `isolation: 'worktree'`，就在一个独立的 git worktree 中运行。这防止并发冲突。

**第三步：限制工具**。根据 Agent 定义中的 `tools` 和 `disallowedTools` 过滤工具列表。Explore Agent 拿到的工具列表里没有 Edit 和 Write。

### 3.1 Fork 的优势

为什么不用线程而用 Fork？因为 **Fork 更安全**。子 Agent 的所有操作都在隔离环境里，不会意外修改主 Agent 的工作状态。任何一个子 Agent 崩溃，都不会影响主 Agent。

### 3.2 上下文传递

```
主 Agent 上下文
├── System Prompt（部分传递）
├── 工具列表（按 Agent 定义过滤）
├── 记忆（按 memory scope 决定）
└── MCP Servers（可继承可添加）
```

主 Agent 的上下文是「只读的参考」，子 Agent 基于它工作，但不会修改它。

## 四、工具解析：Agent 能用什么

每个 Agent 能用什么工具，是由它的定义决定的。这是一种**白名单 + 黑名单**的机制。

```typescript
// agentToolUtils.ts
function resolveAgentTools(agent: AgentDefinition): Tools {
  // 1. 从主工具列表开始（全部工具）
  let tools = getMainTools()
  
  // 2. 应用 allowedTools 过滤器（白名单）
  if (agent.tools) {
    tools = tools.filter(t => agent.tools.includes(t.name))
  }
  
  // 3. 应用 disallowedTools 过滤器（黑名单）
  if (agent.disallowedTools) {
    tools = tools.filter(t => !agent.disallowedTools.includes(t.name))
  }
  
  return tools
}
```

过滤的顺序很重要：**先白名单再黑名单**。如果你同时指定了 `tools: ['Read', 'Edit']` 和 `disallowedTools: ['Edit']`，最终结果只有 `Read`。

这种设计让 Agent 权限的控制变得非常灵活。你可以默认开放所有工具（不指定 `tools`），然后禁止危险操作（`disallowedTools: ['Bash']`）；或者默认禁止所有工具（`tools: ['Read']`），然后按需开放。

## 五、记忆隔离：Agent 的独立记忆空间

每个 Agent 可以有自己的记忆空间，这解决了「Agent 之间记忆污染」的问题。

Claude Code 有三种记忆范围：

**user 记忆**存放在用户级别，所有 Agent 共享。

**project 记忆**存放在项目级别，项目成员共享。

**local 记忆**最特殊——它是**会话级别**的，只在当前会话内有效，会话结束就消失。

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

local 记忆是 Claude Code 的创新。比如你问主 Agent：「刚才 Explore 找到了什么？」主 Agent 不会记得，因为它没有访问 Explore 的 local 记忆。但如果你把 Explore 改成用 project 记忆，主 Agent 就能看到探索结果了。

## 六、Agent 协作模式

多个 Agent 怎么配合工作？Claude Code 支持三种协作模式。

### 6.1 主从模式

最常见的模式。主 Agent 负责任务分解，把子任务分发给专门的 Agent，然后汇总结果。

```
用户 → 主 Agent → 子 Agent (Explore)
                ↓
           子 Agent (Plan)
                ↓
           主 Agent 汇总结果
```

### 6.2 并行模式

多个子 Agent 同时工作，主 Agent 等待所有结果后继续。

```
主 Agent → [Agent A] ← 同时运行
         → [Agent B] ←
         → [Agent C] ←
```

适合「独立任务并行执行」的场景。比如同时探索三个不同的代码模块。

### 6.3 验证模式

```
主 Agent → 完成任务
         → 触发验证 Agent
         → 验证结果
         → 通过/失败
```

专门的验证 Agent（VERIFICATION_AGENT）检查主 Agent 的输出是否正确。

## 七、设计思想：专业化与隔离

Claude Code 的 Agent 架构体现了三个核心原则：

**专业化原则**：每个 Agent 只做一件事。Explore 专攻搜索，Plan 专攻规划，Verification 专攻验证。专业化的 Agent 比通用 Agent 更容易优化——你可以单独调优 Explore，不用担心影响其他功能。

**隔离原则**：工具隔离（Explore 不能写文件）、记忆隔离（local 记忆不污染全局）、工作目录隔离（worktree 模式独立 git 状态）。隔离让错误不会扩散，让并发成为可能。

**组合原则**：简单积木可以组合出复杂系统。主 Agent + Explore = 带探索能力的助手；主 Agent + Plan = 带规划能力的助手；主 Agent + Explore + Plan = 全能助手。

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
