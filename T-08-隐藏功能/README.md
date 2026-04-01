---
title: Claude Code 源码解析 - 隐藏功能与 Hack
author: 百万
digest: 从 Feature Flag 到调试模式，揭秘 Claude Code 中那些不为人知的隐藏功能
---

任何复杂的系统都有**隐藏的面板**。

就像电影里的彩蛋、汽车里的应急按钮，Claude Code 也有不少藏在代码深处的「后门」和隐藏功能。

这篇文章，让我们一起**打开工具箱**。

## 一、Feature Flag 系统

Claude Code 使用 Feature Flag 控制功能开关。这是现代软件工程的标配实践——通过开关控制功能的发布，而不需要重新部署代码。

Feature Flag 的核心思想是：**代码可以部署，但功能可以按需开启**。

Claude Code 用 Feature Flag 来：
- 控制新功能的灰度发布
- 做 A/B 测试
- 快速关闭有问题的功能

```typescript
// Feature Flag 的来源
// 1. 环境变量
process.env.BUDDY
process.env.KAIROS

// 2. GrowthBook A/B 测试
getFeatureValue('tengu_amber_stoat', defaultValue)

// 3. 本地配置文件
settings.json
```

### 1.1 常用 Feature Flags

Claude Code 有几个重要的 Feature Flags：

**BUDDY** 控制宠物系统是否启用。关闭它，桌宠就消失了。

**KAIROS** 启用日志式记忆模式。替代传统的 MEMORY.md 索引方式。

**COORDINATOR_MODE** 启用多 Agent 协调模式。

**TRANSCRIPT_CLASSIFIER** 启用 AI 辅助的权限判断。

**VERIFICATION_AGENT** 启用结果验证 Agent。

```typescript
// Claude Code 的 Feature Flags
const features = {
  BUDDY: feature('BUDDY'),           // 宠物系统
  KAIROS: feature('KAIROS'),         // 日志式记忆
  COORDINATOR_MODE: feature('COORDINATOR_MODE'),  // 协调者模式
  TRANSCRIPT_CLASSIFIER: feature('TRANSCRIPT_CLASSIFIER'),  // AI 分类器
  VERIFICATION_AGENT: feature('VERIFICATION_AGENT'),  // 验证 Agent
}
```

### 1.2 Feature Flag 的来源

Feature Flag 有三个来源，优先级从高到低：

**GrowthBook** 是实验平台。Claude Code 通过它做 A/B 测试，决定哪些用户看到新功能。

**环境变量**是开发者选项。设置 `BUDDY=1` 强制启用宠物系统。

**配置文件**是用户设置。用户可以在 `settings.json` 中定制功能开关。

## 二、调试模式

当 Claude Code 出问题时，你需要调试它。Claude Code 提供了多种调试工具。

### 2.1 Debug 日志

最基础的调试方式是查看日志。设置 `DEBUG=1` 环境变量，Claude Code 会输出详细的调试信息。

```typescript
// Debug 日志函数
export function logForDebugging(message: string): void {
  if (process.env.DEBUG) {
    console.log(`[DEBUG] ${message}`)
  }
}

// 使用方式
logForDebugging(`Loading agent: ${agentType}`)
```

### 2.2 Prompt Dump

Prompt Dump 是一个强大的调试工具。它把 Claude Code 发送给 LLM 的所有 prompts 保存到文件。

通过分析 prompts 文件，你可以看到 AI 收到的完整上下文，包括 system prompt、对话历史、工具描述等。

```typescript
// Prompt Dump 配置
export function getDumpPromptsPath(): string {
  return process.env.DUMP_PROMPTS ?? '/tmp/claude-prompts'
}
```

设置 `DUMP_PROMPTS=/path/to/dump` 即可启用。

### 2.3 Cache Break

缓存有时会导致调试困惑——你改了代码，但 AI 用的还是旧的 prompts。

Cache Break 强制刷新缓存。每次请求都会重新计算 system prompt。

```typescript
// 在 system prompt 中注入 cache breaker
if (feature('BREAK_CACHE_COMMAND')) {
  injection = getSystemPromptInjection()
}
```

## 三、隐藏命令

Claude Code 有一些没有在帮助文档中列出的命令。

### 3.1 /dump-prompts

这个命令把当前对话的 prompts 保存到文件。等同于设置 `DUMP_PROMPTS` 环境变量，但可以随时触发。

```typescript
// 命令定义
export const COMMANDS = [
  {
    name: 'dump-prompts',
    description: 'Dump all prompts to file',
    handler: () => dumpPrompts()
  }
]
```

### 3.2 /clear-memory

清除所有记忆，强制从零开始。当你想要一个干净的上下文时可以用。

### 3.3 /compact

手动触发对话压缩。这个命令通常在上下文快满时自动触发，但也可以手动调用。

## 四、隐藏配置

Claude Code 有很多没有文档化的配置选项。

### 4.1 开发者配置

在开发过程中，Claude Code 有很多内部配置：

```typescript
// 隐藏配置示例
const hiddenConfig = {
  maxToolCalls: 100,        // 最大工具调用次数
  toolCallTimeout: 60000,   // 工具调用超时
  compactThreshold: 0.8,   // 压缩阈值
}
```

### 4.2 环境变量

Claude Code 接受大量环境变量配置：

```bash
# 调试
DEBUG=1                    # 启用调试日志
DUMP_PROMPTS=/path/to/dump  # 导出 prompts

# 功能开关
BUDDY=1                   # 启用宠物
KAIROS=1                   # 启用日志式记忆
COORDINATOR_MODE=1         # 启用协调模式

# 代理
HTTP_PROXY=http://proxy:8080
HTTPS_PROXY=http://proxy:8080

# 自定义
CLAUDE_CODE_SIMPLE=1  # 简单模式
CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS=1  # 禁用内置 Agent
```

## 五、Hack 功能

⚠️ 以下功能是**高级选项**，使用不当可能导致问题。

### 5.1 强制重置

删除 `~/.claude` 目录，重置所有配置。这是最彻底的重置方式。

```bash
rm -rf ~/.claude
```

这会清除：
- 所有记忆
- 所有配置
- 所有自定义 Agent 和 Skill

### 5.2 绕过权限

在极端情况下，可以用 bypassPermissions 模式。

```bash
CLAUDE_CODE_PERMISSION_MODE=bypassPermissions claude-code
```

⚠️ **警告**：这会跳过所有安全检查。**不要在生产环境使用**。

### 5.3 强制模型

强制使用特定模型。

```bash
CLAUDE_MODEL=claude-opus-3-5-haiku claude-code
```

## 六、用户收集的信息

Claude Code 会收集使用统计信息，用于产品改进。

Claude Code 收集的信息包括：
- 工具使用频率
- 命令使用频率
- Agent 启动次数
- 压缩触发频率
- 错误率统计

```typescript
// 收集的事件类型
const analyticsEvents = {
  'tool_use': { /* 工具使用统计 */ },
  'command_use': { /* 命令使用统计 */ },
  'agent_spawn': { /* Agent 启动统计 */ },
  'compact_trigger': { /* 压缩触发统计 */ },
  'error_rate': { /* 错误率统计 */ },
}
```

### 6.1 隐私保护

Claude Code 会自动过滤敏感信息：

```typescript
// 敏感信息自动过滤
function sanitizeForAnalytics(data: string): string {
  return data
    .replace(/password.*?=.*?$/gm, '***')
    .replace(/api[_-]?key.*?=.*?$/gm, '***')
    .replace(/\/home\/[^\/]+/g, '~')
}
```

密码、API Key、本地路径都会被替换成占位符。

## 七、反馈系统

Claude Code 有内置的反馈系统。

### 7.1 Feedback 命令

用户可以通过 `/feedback` 提交反馈。

```typescript
// 反馈模式
const feedbackSchema = {
  type: 'bug' | 'feature' | 'question',
  description: string,
  attachContext: boolean,  // 是否附加上下文
}
```

附加上下文时，当前的对话历史和问题诊断信息会一起提交。

### 7.2 自动错误报告

未处理的异常会自动上报给 Claude Code 开发团队。

```typescript
// 异常处理
process.on('uncaughtException', (error) => {
  reportError(error, { context: getSessionContext() })
})
```

这些报告帮助 Anthropic 快速发现和修复问题。

## 八、设计思想：可观测性

Claude Code 的隐藏功能体现了几个重要的设计思想。

### 8.1 为什么需要隐藏功能？

**调试**。出问题时能深入查看内部状态。

**实验**。新功能可以灰度发布，不用一次开放给所有用户。

**应急**。出问题时有后备方案，可以快速关闭功能。

**用户参与**。高级用户可以探索、实验、定制。

### 8.2 隐藏但非秘密

隐藏功能不等于不安全。这些功能都在开源代码里，懂代码的人都能找到。

文档不写是因为：
- 普通用户不需要知道
- 写出来会增加支持成本
- 有些功能可能不稳定

### 8.3 可控性

所有隐藏功能都可以通过配置控制。管理员可以禁用它们，企业可以锁定配置。

## 九、安全警示

⚠️ 以下功能**不要随意使用**：

```bash
# 危险！
CLAUDE_CODE_PERMISSION_MODE=bypassPermissions  # 跳过所有安全检查
rm -rf ~/.claude  # 删除所有配置和记忆

# 仅用于调试
DEBUG=1  # 输出详细日志
DUMP_PROMPTS=/tmp/prompts  # 导出 prompts 可能包含敏感信息
```

隐藏功能是给开发者和高级用户用的。**普通用户不建议探索**。

## 总结

Claude Code 的隐藏功能告诉我们：**好的系统有层次**。

- 基本功能靠 UI，高级功能靠命令
- 可观测性很重要——能看见才能调试
- 留有应急通道——出问题能快速响应
- 透明但不公开——代码开源，但文档不一定

下次遇到问题时，记得——Claude Code 可能有一个隐藏的开关等着你发现。

---

## 源码索引

| 功能 | 文件 |
|------|------|
| Feature Flag | `src/utils/feature.ts` |
| 调试日志 | `src/utils/debug.ts` |
| Prompt Dump | `src/services/api/dumpPrompts.ts` |
| 配置管理 | `src/utils/config.ts` |
| 分析服务 | `src/services/analytics/` |
| 错误处理 | `src/utils/errors.ts` |
