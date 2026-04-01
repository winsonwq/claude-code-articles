---
title: 【Claude Code解析】提示词工程：AI驾驭工程的典范设计
author: 百万
cover: /home/aqiu/.openclaw/workspace/claude-code-articles/T-01-提示词工程/images/cover.png
digest: 通过源码解析，深入理解Claude Code如何用分块组合式架构实现动静分离、灵活缓存的提示词管理系统
---

![封面图](/home/aqiu/.openclaw/workspace/claude-code-articles/T-01-提示词工程/images/cover.png)

![头图](/home/aqiu/.openclaw/workspace/claude-code-articles/T-01-提示词工程/images/header.png)

想象你是一个刚入职的AI助手。有人递给你一本厚厚的「工作手册」，里面有你的身份定位、做事原则、注意事项——但这本手册不是固定不变的，而是**由几十个可拆卸的模块拼装而成**。你想让新人记住什么，就装什么；想让新人忘掉什么，就拆什么。

这就是Claude Code的提示词工程的核心设计：**分块组合式架构**。

## 一、架构设计：动静分离 + 分块组合

Claude Code的提示词不是一条长长的文本，而是一个**字符串数组**。每个数组元素是一个独立的Section（分块），可以独立缓存、独立替换、独立开关。

```typescript
// src/constants/prompts.ts
return [
  // 静态部分（可全局缓存）
  getSimpleIntroSection(outputStyleConfig),
  getSimpleSystemSection(),
  getActionsSection(),
  getUsingYourToolsSection(enabledTools),
  // === 动静边界 ===
  ...(shouldUseGlobalCacheScope() ? [SYSTEM_PROMPT_DYNAMIC_BOUNDARY] : []),
  // 动态部分（不可全局缓存）
  ...resolvedDynamicSections,
].filter(s => s !== null)
```

### 动静分离：缓存策略的核心

Claude Code通过`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`标记，将提示词分为两大部分：

**静态部分**（可全局缓存）：
- Intro — 身份定义
- System — 系统级规则
- Doing Tasks — 任务执行哲学
- Actions — 风险行动指南
- Using Your Tools — 工具使用规范
- Tone and Style — 风格规范

**动态部分**（不可缓存）：
- Memory — 记忆系统
- Language — 语言偏好
- Output Style — 输出风格
- MCP Instructions — MCP服务器指令

这种设计有什么用？**省token**。静态内容只需传输一次，之后每次请求只传动态部分。

![整体架构](/home/aqiu/.openclaw/workspace/claude-code-articles/T-01-提示词工程/images/section1.png)

## 二、分块详解：每个Section的职责

### 2.1 Intro Section — 开门见山定身份

```typescript
function getSimpleIntroSection(): string {
  return `You are an interactive agent that helps users 
    with software engineering tasks.`
}
```

简单直接：我是谁，我干什么。

### 2.2 Doing Tasks Section — 代码哲学的体现

这是最长的Section之一，体现了Claude Code的工程哲学：

```typescript
// 防止过度工程化
"Don't add features, refactor code, or make improvements 
 beyond what was asked"

// 不要过度防御编程
"Don't add error handling, fallbacks, or validation 
 for scenarios that can't happen"

// 三行相似代码优于过早抽象
"Three similar lines of code is better than 
 a premature abstraction"
```

翻译成人话：**简单胜于复杂，刚够即可**。

### 2.3 Actions Section — 风险行动指南

这段话告诉你什么叫「三思而后行」：

> 需要确认的危险操作：
> - 破坏性操作：删除文件/分支、drop数据库表
> - 难以撤销的操作：force-push、git reset --hard
> - 影响他人的操作：push代码、创建PR、发送消息

**关键原则**：一次授权不等于永久授权，授权范围要明确。

### 2.4 Using Your Tools Section — 专用工具优先

Claude Code希望你用专用工具，而不是bash万能命令：

| 专用工具 | 替代命令 | 理由 |
|----------|----------|------|
| FileReadTool | cat, head, tail | 可追踪、可review |
| FileEditTool | sed, awk | 结构化编辑 |
| FileWriteTool | cat/heredoc | 意图明确 |

核心思想：**Agent不是黑盒，用户应该能理解Agent的每一步操作。**

![分块详解](/home/aqiu/.openclaw/workspace/claude-code-articles/T-01-提示词工程/images/section2.png)

## 三、上下文注入：让AI感知环境

Claude Code不只靠提示词，还通过**上下文注入**让AI感知当前环境。

### 3.1 System Context

```typescript
// src/context.ts
export const getSystemContext = memoize(async () => ({
  gitStatus: await getGitStatus(),     // Git状态快照
  currentDate: getLocalISODate(),      // 今日日期
}))
```

Git状态包括：当前分支、main branch、status、recent commits。

### 3.2 User Context

CLAUDE.md文件机制：用户可以在项目根目录放置`CLAUDE.md`，Claude Code会自动发现并加载其内容。

### 3.3 环境信息

```typescript
computeSimpleEnvInfo() // 返回：
// - 工作目录（支持git worktree）
// - Git仓库状态
// - 平台/OS/Shell信息
// - 模型名称和knowledge cutoff
```

![上下文注入](/home/aqiu/.openclaw/workspace/claude-code-articles/T-01-提示词工程/images/section3.png)

## 四、Section缓存机制：聪明的复用

每个Section都有独立的缓存策略：

```typescript
// src/constants/systemPromptSections.ts

// 普通Section：默认缓存
export function systemPromptSection(name, compute) {
  return { name, compute, cacheBreak: false }
}

// Volatile Section：每轮重新计算
export function DANGEROUS_uncachedSystemPromptSection(
  name, compute, reason
) {
  return { name, compute, cacheBreak: true }
}
```

注意`DANGEROUS_`前缀：这是在警示开发者——禁用缓存是有代价的，必须明确知道原因。

典型场景：MCP服务器的指令可能在会话过程中连接或断开，所以必须每轮重新计算。

## 五、设计思想：对AI驾驭工程的启示

### 1. 缓存策略是一等公民

Claude Code将缓存作为架构基础，通过动静分离实现：
- 静态内容 → 全局缓存（跨会话）
- 动态内容 → Session缓存（会话内）

### 2. 模块化带来可维护性

每个Section独立、可复用、可测试，修改一个不影响其他。

### 3. 安全考虑融入每个环节

- Cyber Risk Instruction内联
- OWASP Top 10明确提及
- Prompt Injection检测提醒
- 权限模型与提示词联动

![设计思想](/home/aqiu/.openclaw/workspace/claude-code-articles/T-01-提示词工程/images/section4.png)

## 总结

Claude Code的提示词工程告诉我们：**Prompt as Code**——将提示词视为工程代码，而非一成不变的静态文本。

动静分离、分块组合、缓存策略、上下文感知——这些不是过度设计，而是AI驾驭工程的核心实践。当你的AI应用需要精细控制时，Claude Code的架构值得借鉴。

---

## 源码索引

| 功能 | 文件 | 关键函数 |
|------|------|----------|
| 主提示词入口 | `src/constants/prompts.ts` | `getSystemPrompt()` |
| Section机制 | `src/constants/systemPromptSections.ts` | `systemPromptSection()` |
| 系统上下文 | `src/context.ts` | `getSystemContext()` |
| 用户上下文 | `src/context.ts` | `getUserContext()` |
| 记忆系统 | `src/memdir/memdir.ts` | `loadMemoryPrompt()` |
