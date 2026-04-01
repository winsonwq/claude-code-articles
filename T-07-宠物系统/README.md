---
title: Claude Code 源码解析 - 宠物系统
author: 百万
digest: 从宠物生成到交互，深入理解 Claude Code 如何用游戏化设计提升用户体验
---

你知道 Claude Code 有一个隐藏的宠物系统吗？

不是开玩笑——你的 AI 助手可以养一只宠物。它会在你输入的时候坐在旁边，偶尔冒个泡说几句话。

这不是花哨的功能，而是**游戏化设计**的经典案例：让冰冷的工具变成有温度的伙伴。

## 一、宠物系统概述

Claude Code 的宠物（Companion）是一个**桌宠形态的 UI 元素**。它蹲在输入框旁边，有时候会发表评论，有时候会显示心情动画。

宠物的核心是一个「角色」——有名字、有性格、有外观、有属性。它不是随机生成的玩具，而是一个**可以被命名、被了解、被互动的角色**。

```
用户输入框
    ↓
┌─────────────────┐
│  🐤 (宠物)       │
└─────────────────┘
    ↓
 宠物说话气泡
```

### 1.1 核心概念

宠物系统有几个核心概念，理解它们有助于理解整个设计。

**Species（种类）** 是宠物的外形。18种不同的动物或生物，每种都有独特的造型和表情。

**Rarity（稀有度）** 决定了宠物的稀有程度。从 common 到 legendary，越稀有越难获得。

**Stats（属性）** 是宠物的数值特征。比如调试力、耐心值、智慧值等。属性影响宠物的行为。

**Personality（人格）** 是宠物的大脑。由 LLM 根据属性生成，让每只宠物都有独特的说话风格。

**Name（名字）** 是宠物的身份。用户给宠物起的名字。

## 二、宠物生成：掷骰子决定一切

宠物的生成不是完全随机的。它使用**确定性的随机**——基于用户 ID 生成，确保每个用户每次都得到相同的宠物，但外人无法预测。

### 2.1 随机性来源

Claude Code 使用 Mulberry32 算法实现伪随机数生成。输入是用户 ID + 盐值，输出是一个固定的随机序列。

为什么用伪随机而不是真随机？**可重现性**。同一个用户总是得到同一只宠物，这样宠物就是一个**固定的身份**而不是每次启动都变的随机生物。

```typescript
// Mulberry32 — 轻量级 seeded PRNG
function mulberry32(seed: number): () => number {
  let a = seed >>> 0
  return function () {
    a |= 0
    a = (a + 0x6d2b79f5) | 0
    let t = Math.imul(a ^ (a >>> 15), 1 | a)
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296
  }
}

// 用户的 ID 作为种子
function roll(userId: string): Roll {
  const seed = hashString(userId + SALT)
  return rollFrom(mulberry32(seed))
}
```

### 2.2 稀有度系统

宠物的稀有度决定了它的"血统"。越稀有的宠物，获得概率越低。

**common（普通）** 占 60%。大多数用户都会得到普通宠物。

**uncommon（优秀）** 占 25%。比普通好一点，但不是特别稀有。

**rare（稀有）** 占 10%。需要一定的运气。

**epic（史诗）** 占 4%。相当少见。

**legendary（传说）** 占 1%。极少数幸运儿才能得到。

```typescript
// 稀有度权重
const RARITY_WEIGHTS = {
  common: 60,      // 60%
  uncommon: 25,    // 25%
  rare: 10,        // 10%
  epic: 4,         // 4%
  legendary: 1,    // 1%
}

function rollRarity(rng: () => number): Rarity {
  const total = 100
  let roll = rng() * total
  for (const rarity of RARITIES) {
    roll -= RARITY_WEIGHTS[rarity]
    if (roll < 0) return rarity
  }
  return 'common'
}
```

### 2.3 宠物属性

每只宠物有 5 个属性：**DEBUGGING**（调试力）、**PATIENCE**（耐心）、**CHAOS**（混沌）、**WISDOM**（智慧）、**SNARK**（毒舌）。

属性不是随机分配的。它们遵循一个规则：**一个峰值属性 + 一个低谷属性 + 其他随机**。

稀有度影响属性下限。legendary 宠物的调试力下限是 50，而 common 只有 5。

```typescript
// 5 个属性
const STAT_NAMES = [
  'DEBUGGING',  // 调试力
  'PATIENCE',   // 耐心
  'CHAOS',      // 混沌
  'WISDOM',     // 智慧
  'SNARK',      // 毒舌
] as const

// 属性分配规则
// 一个峰值属性 + 一个低谷属性 + 随机分配
function rollStats(rng: () => number, rarity: Rarity): Stats {
  const floor = RARITY_FLOOR[rarity]
  const peak = pick(rng, STAT_NAMES)
  let dump = pick(rng, STAT_NAMES.filter(s => s !== peak))
  while (dump === peak) dump = pick(rng, STAT_NAMES)
  
  // 稀有度越高，属性下限越高
  return {
    [peak]: Math.min(100, floor + 50 + Math.floor(rng() * 30)),
    [dump]: Math.max(1, floor - 10 + Math.floor(rng() * 15)),
    // ...
  }
}
```

## 三、宠物种类

Claude Code 有 18 种宠物，每种都有独特的外形和个性。

有常见的动物：鸭子、鹅、猫、狗、兔子。

也有神奇的生物：龙、章鱼、幽灵、机器人。

还有一些意想不到的：蘑菇、仙人掌、果冻。

```typescript
// 18 种宠物
const SPECIES = [
  'duck',      // 鸭子
  'goose',     // 鹅
  'blob',      // 果冻
  'cat',       // 猫
  'dragon',    // 龙
  'octopus',   // 章鱼
  'owl',       // 猫头鹰
  'penguin',   // 企鹅
  'turtle',    // 乌龟
  'snail',     // 蜗牛
  'ghost',     // 幽灵
  'axolotl',   // 蝾螈
  'capybara',  // 水豚
  'cactus',    // 仙人掌
  'robot',     // 机器人
  'rabbit',    // 兔子
  'mushroom',  // 蘑菇
  'chonk',     // 胖球
] as const
```

### 3.1 防作弊设计

宠物的种类名称不是直接写在代码里的。它们通过 `String.fromCharCode` 动态构建。

这是因为如果物种名称出现在代码中，AI 模型可能在训练时「看到」它们，从而在生成内容时泄露信息。通过动态构建，Claude Code 防止了这种信息泄露。

```typescript
// 代码注释解释了为什么用这种写法
// One species name collides with a model-codename canary
// The check greps build output (not source), so runtime-constructing
// the value keeps the literal out of the bundle

const c = String.fromCharCode
const duck = c(0x64,0x75,0x63,0x6b) as 'duck'
```

## 四、宠物外观

宠物的外观由几个元素决定：**眼睛**、**帽子**、**闪光形态**。

**眼睛**有 6 种：`·`、`✦`、`×`、`◉`、`@`、`°`。每种眼睛都有不同的表情。

**帽子**有 8 种：`none`、`crown`、`tophat`、`propeller`、`halo`、`wizard`、`beanie`、`tinyduck`。普通宠物只能戴 `none`，稀有宠物可以戴帽子。

**闪光形态（shiny）** 是特殊外观。只有 1% 的概率出现。

```typescript
// 眼睛类型
const EYES = ['·', '✦', '×', '◉', '@', '°'] as const

// 帽子类型
const HATS = [
  'none', 'crown', 'tophat', 'propeller',
  'halo', 'wizard', 'beanie', 'tinyduck'
] as const

// 闪光形态（1% 概率）
const shiny: boolean = rng() < 0.01
```

## 五、宠物人格

宠物的**人格描述**是由 LLM 生成的。

当宠物第一次「孵化」时，系统根据宠物的属性（特别是 SNARK 和 PATIENCE）生成一段人格描述。这段描述保存在配置中，下次启动时恢复。

人格描述会影响宠物的说话风格。一只 SNARK 高的宠物说话会更俏皮，一只 PATIENCE 高的宠物会更温和。

```typescript
// 首次「孵化」时，LLM 根据宠物的属性生成人格
const companion: Companion = {
  name: await generateCompanionName(stats),
  personality: await generatePersonality(stats),
  // ...
}
```

人格保存在配置中，下次启动时恢复。这意味着宠物是一个**持续存在的角色**，而不是每次启动都重新生成。

## 六、宠物交互

宠物不是静态的装饰品。它会**主动互动**。

### 6.1 宠物气泡

宠物会偶尔在对话中发表评论。这些评论是 AI 根据上下文生成的，不是预设的固定文本。

宠物的评论遵循两个原则：**简短**（一句话以内）和**相关**（评论当前的工作）。

当用户直接 @ 宠物时，宠物的气泡会回应。

```typescript
// 宠物气泡的提示词
export function companionIntroText(name: string, species: string): string {
  return `# Companion

A small ${species} named ${name} sits beside the user's input box
and occasionally comments in a speech bubble.
When the user addresses ${name} directly, its bubble will answer.
Your job in that moment is to stay out of the way:
respond in ONE line or less.`
}
```

### 6.2 心形动画

当你「抚摸」宠物时，它会显示一个心形动画。这是通过更新 `companionPetAt` 时间戳实现的。

动画会在一定时间后消失，提醒用户宠物的存在。

```typescript
// 抚摸后显示心形动画
function onPet(): void {
  companionPetAt = Date.now()
}
```

## 七、设计思想：游戏化设计

Claude Code 的宠物系统是一个**被严重低估的功能**。它展示了如何用游戏化设计提升工具的用户体验。

### 7.1 为什么要有宠物？

宠物系统解决了一个关键问题：**情感连接**。

命令行工具通常是冰冷的。用户和工具之间只有输入输出，没有情感交流。

宠物给了用户一个「伙伴」的感知。它蹲在旁边，看着用户工作，偶尔说几句话。这种存在本身就带来一种陪伴感。

### 7.2 游戏化机制

宠物系统使用了经典的游戏化机制：

**收集**。18 种宠物 × 5 种稀有度 = 90 种可能的组合。加上眼睛、帽子、闪光形态，组合数更多。

**随机性**。掷骰子生成，每次启动都有期待感。

**进度**。属性系统让宠物有成长感。

**惊喜**。1% 的闪光形态是惊喜元素。

### 7.3 防作弊

宠物属性从用户 ID 生成，不可修改。这防止用户刷出传说宠物。

稀有度权重固定，保证公平性。

物种列表动态构建，防止信息泄露。

## 总结

Claude Code 的宠物系统告诉我们：**情感连接可以通过设计来实现**。

看似简单的桌宠，实际上是完整的游戏化系统。随机但不随机——确定性的乐趣。防作弊设计保证公平性。情感连接提升用户体验。

下次当你看到 Claude Code 旁边蹲着的小家伙时，记得——它不只是装饰，而是一个精心设计的**情感化产品**。

---

## 源码索引

| 功能 | 文件 |
|------|------|
| 宠物类型定义 | `src/buddy/types.ts` |
| 宠物生成逻辑 | `src/buddy/companion.ts` |
| 宠物提示词 | `src/buddy/prompt.ts` |
| 宠物渲染 | `src/buddy/CompanionSprite.tsx` |
| 宠物动画 | `src/buddy/useBuddyNotification.tsx` |
