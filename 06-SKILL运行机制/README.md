---
title: Claude Code 源码解析 - SKILL 运行机制
author: 百万
digest: 从 Skill 定义到执行流程，深入理解 Claude Code 如何让 AI 掌握新技能
---

想象你是一个刚入职的程序员。现在让你去修空调，你一脸懵——你会写代码，但不会修空调。

但如果有人给你一本《空调维修手册》，你就能照着手册修了。

Claude Code 的 **Skill 系统**就是这本**技能手册**：让 AI 能快速掌握新能力，不需要重新训练模型。

## 一、Skill 架构：Markdown 即技能

Claude Code 的 Skill 和 Agent 一样，用 **Markdown 写配置**。这延续了 Claude Code「配置即代码」的设计哲学。

为什么用 Markdown？因为 Markdown 本身就是一种**结构化文档格式**。它的 frontmatter 可以放元数据，正文可以放指令和示例。天然适合定义技能。

一个 Skill 就是一个 Markdown 文件。文件头部是 frontmatter，定义技能的名字、描述；正文是 instruction，告诉 AI 这个技能怎么用。

```markdown
---
name: my-skill
description: 当你需要 XXX 时使用这个技能
instruction: 执行步骤...
---

# My Skill

这个技能可以帮助你...
```

### 1.1 Skill 与 Agent 的区别

虽然 Skill 和 Agent 都用 Markdown 定义，但它们有本质区别。

**Agent 是独立的工作单元**。它有自己独立的对话循环、独立的上下文。像一个独立的员工，被派去完成一个完整的子任务。

**Skill 是技能的增强**。它运行在当前对话中，不创建独立的对话循环。像员工学的一门技能，让员工能做新的事，但员工还是员工。

| | Agent | Skill |
|--|-------|-------|
| **用途** | 独立的工作单元 | 增强工具能力 |
| **粒度** | 粗粒度（完整的子任务） | 细粒度（单一功能） |
| **调用方式** | `/agent` 或 Agent tool | `/my-skill` 或 Skill tool |
| **运行模式** | 独立对话循环 | 在当前对话中执行 |

简单说：**Agent 是独立员工，Skill 是员工学的技能**。

## 二、Skill 定义结构

从代码层面看，Skill 的定义和 Agent 非常相似，但更轻量。

Skill 最核心的三个属性是：**name**、**description**、**instruction**。名字是唯一标识，描述说明什么时候用，指令说明怎么用。

可选属性用来控制 Skill 的行为：`tools` 和 `disallowedTools` 控制可用工具，`arguments` 定义接受的参数，`subskills` 让技能可以包含子技能。

```typescript
// src/skills/loadSkillsDir.ts
type SkillDefinition = {
  name: string              // 技能名称（唯一标识）
  description: string       // 何时使用
  instruction: string       // 执行步骤
  category?: string         // 分类
  arguments?: SkillArgument[]  // 参数定义
  subskills?: SkillDefinition[]  // 子技能
  
  // 执行控制
  maxTurns?: number         // 最大回合数
  effort?: EffortValue      // 努力程度
  tools?: string[]          // 允许使用的工具
  disallowedTools?: string[]  // 禁止使用的工具
  
  // 来源追踪
  source: 'skills' | 'plugin' | 'managed' | 'bundled'
}
```

Skill 和 Agent 的一个关键区别是：**Skill 没有独立的 system prompt**。Skill 的指令（instruction）只是追加到当前对话的文本，不会替换 system prompt。

## 三、Skill 加载：多来源聚合

Claude Code 从多个来源加载 Skill，就像一个图书馆从不同渠道采购书籍。

**内置 Skill（bundled）** 是 Claude Code 官方提供的，比如 `claude-code-guide`，教用户怎么用 Claude Code。

**插件 Skill（plugin）** 是第三方插件提供的。

**托管 Skill（managed）** 是企业管理员统一管理的。

**用户 Skill（skills）** 是用户自己创建的，存放在 `~/.claude/skills/` 或 `<project>/.claude/skills/`。

```typescript
// Skill 来源优先级
const SKILL_SOURCES = [
  'bundled',    // 内置技能
  'plugin',     // 插件提供
  'managed',    // 托管配置
  'skills',     // 用户自定义
]
```

多个来源的同名 Skill 会合并，后加载的覆盖先加载的。

### 3.1 内置 Skill

Claude Code 自带一些内置 Skill，帮助用户上手：

`claude-code-guide` 是交互式教程，带你了解 Claude Code 的基本用法。

这些 Skill 是 Claude Code 的一部分，开箱即用。

### 3.2 用户 Skill

用户可以创建自己的 Skill。Skill 文件存放在特定目录：

```
~/.claude/skills/          # 用户级别，所有项目可见
<project>/.claude/skills/   # 项目级别，只有该项目可见
```

创建一个 Skill 只需要一个 Markdown 文件。这就是 Claude Code 的设计哲学：**门槛低，能力深**。

## 四、Skill 执行流程

当用户调用 `/my-skill` 时，Claude Code 经历以下步骤：

**解析**：识别技能名称和参数。

**加载**：从磁盘或缓存读取 Skill 定义。

**参数验证**：如果 Skill 定义了参数，验证用户提供的参数是否合法。

**执行**：把 instruction 追加到当前对话，让 AI 按指令执行。

**返回**：Skill 执行完成后，返回结果给用户。

```
用户调用 /my-skill
       ↓
  解析 Skill 名称
       ↓
  加载 Skill 定义
       ↓
  验证参数（如果有）
       ↓
  在当前对话执行 instruction
       ↓
  返回执行结果
```

### 4.1 解析与参数替换

Skill 可以接受参数。比如 `/deploy --env production --service api`。

参数用 `{{variable}}` 语法在 instruction 中标记。执行时，系统会用实际参数替换这些占位符。

```typescript
// Skill 可以接受参数
// /deploy --env production --service api

function parseSkillArgs(skill: SkillDefinition, rawArgs: string) {
  // 1. 解析参数名
  const argNames = parseArgumentNames(skill.arguments)
  
  // 2. 替换指令中的 {{arg}}
  const instruction = substituteArguments(skill.instruction, argNames)
  
  return instruction
}
```

### 4.2 子技能调用

复杂的 Skill 可以包含子技能。比如一个 `full-stack-deploy` Skill 可以包含 `build`、`test`、`deploy` 三个子技能。

子技能让 Skill 可以模块化，便于复用和维护。

```typescript
// Skill 可以调用子技能
const skill = {
  name: 'full-stack-deploy',
  subskills: [
    { name: 'build', instruction: '...' },
    { name: 'test', instruction: '...' },
    { name: 'deploy', instruction: '...' },
  ]
}
```

## 五、Skill 与工具的集成

### 5.1 SkillTool

Skill 通过 SkillTool 和系统集成。当用户输入 `/my-skill` 时，实际调用的是 SkillTool。

SkillTool 的 handler 负责加载 Skill 定义并执行它。

```typescript
// src/tools/SkillTool/SkillTool.ts
const SkillTool = {
  name: 'Skill',
  description: 'Execute a skill',
  
  inputSchema: {
    name: 'skill',      // Skill 名称
    arguments: {}       // 可选参数
  },
  
  handler: async (input) => {
    const skill = await loadSkill(input.name)
    return await executeSkill(skill, input.arguments)
  }
}
```

### 5.2 Skill 作为增强

Skill 可以增强现有工具的能力。比如一个 `git-helper` Skill 告诉 AI 怎么用 git 命令。

Skill 不是替代工具，而是**指导 AI 更好地使用工具**。

## 六、MCP Skill：外部系统集成

MCP（Model Context Protocol）Skill 允许调用外部系统的工具。

通过 MCP Skill，AI 可以用自然语言操作外部系统——数据库、API、监控系统等。

```typescript
// MCP Skill 的构建
function buildMcpSkill(server: string, tool: string): SkillDefinition {
  return {
    name: `${server}:${tool}`,
    description: `MCP tool: ${server}/${tool}`,
    instruction: `调用 MCP 服务器 ${server} 的 ${tool} 工具...`,
    source: 'mcp'
  }
}
```

MCP Skill 让 Claude Code 的能力边界变得无限——只要有 MCP 服务器，就能操作任何系统。

## 七、Skill 的执行上下文

### 7.1 上下文隔离

Skill 运行在当前对话的上下文中，但不等于完全共享。

Skill 可以声明自己需要的工具和记忆范围。系统会根据 Skill 的声明，配置它的执行环境。

```typescript
// Skill 执行时的上下文
const skillContext = {
  // 继承主对话的上下文
  systemPrompt: parentContext.systemPrompt,
  
  // Skill 特定的工具限制
  tools: resolveSkillTools(skill),
  
  // Skill 特定的记忆
  memory: skill.memory,
  
  // 隔离的工作目录（可选）
  cwd: skill.isolation ? createIsolatedDir() : parentContext.cwd
}
```

### 7.2 资源限制

Skill 有执行资源限制，防止它占用太多系统资源。

每个 Skill 有最大回合数限制（`maxTurns`）。超过限制会自动停止。

还有 token 预算和超时设置。

```typescript
// Skill 的资源限制
const skillLimits = {
  maxTurns: skill.maxTurns ?? 10,
  maxTokens: skill.maxTokens ?? 5000,
  timeout: skill.timeout ?? 60000,  // 60 秒超时
}
```

## 八、Skill 与 Agent 的选择

什么时候用 Skill，什么时候用 Agent？

**用 Agent** 当你需要：
- 完整的独立工作循环
- 复杂的上下文隔离
- 多个工具协作完成子任务

**用 Skill** 当你需要：
- 单一功能的增强
- 在当前对话中快速执行
- 按步骤指引 AI

```
需要完整的工作循环？→ Agent
需要单一功能增强？→ Skill

需要独立的上下文？→ Agent
需要在当前对话中执行？→ Skill

需要多个工具协作？→ Agent
只需要按步骤执行？→ Skill
```

## 九、设计思想：技能即模块

Claude Code 的 Skill 系统体现了**模块化**的思想。

**可复用性**。一个 Skill 可以被多个 Agent 共享，就像一个人学的技能可以被多个项目使用。

**渐进式能力**。新能力不需要重新训练模型，只需要写一个 Skill。门槛低，迭代快。

**组合性**。复杂任务可以分解为多个 Skill 的组合。简单技能的叠加产生复杂能力。

## 总结

Claude Code 的 Skill 系统告诉我们：**能力不等于模型，能力等于模块**。

- Skill 用 Markdown 定义，门槛低
- Skill 可以组合，复杂能力来自简单技能的叠加
- Skill 有执行上下文隔离，安全可控
- Skill 与 Agent 互补，粒度选择更灵活

如果你想让 AI 掌握新能力，先问：需要独立的子 Agent，还是只是一个 Skill？

---

## 源码索引

| 功能 | 文件 |
|------|------|
| Skill 加载 | `src/skills/loadSkillsDir.ts` |
| 内置 Skill | `src/skills/bundledSkills.ts` |
| Skill 工具 | `src/tools/SkillTool/SkillTool.ts` |
| MCP Skill | `src/skills/mcpSkillBuilders.ts` |
| 参数解析 | `src/utils/argumentSubstitution.ts` |
