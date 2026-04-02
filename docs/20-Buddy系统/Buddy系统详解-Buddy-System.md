# Buddy 伙伴系统详解

## 概述

Buddy 是 Claude Code 的**伙伴/宠物系统**，为用户提供一个可爱的 ASCII art 小伙伴，陪伴在输入框旁边。该系统通过 `/buddy` 命令激活，是一个 feature-gated 功能（需要 `BUDDY` feature flag）。

**注意**：Buddy 系统于 2026 年 4 月 1 日首次上线（愚人节彩蛋），预告窗口为 4 月 1-7 日，之后命令永久可用。

## 核心文件

| 文件 | 说明 |
|------|------|
| [src/buddy/types.ts](src/buddy/types.ts) | 类型定义：物种、稀有度、帽子、属性 |
| [src/buddy/companion.ts](src/buddy/companion.ts) | 核心逻辑：确定性生成、配置读取 |
| [src/buddy/sprites.ts](src/buddy/sprites.ts) | ASCII art 精灵帧动画 |
| [src/buddy/CompanionSprite.tsx](src/buddy/CompanionSprite.tsx) | React 组件：伙伴显示 + 对话气泡 |
| [src/buddy/prompt.ts](src/buddy/prompt.ts) | 提示文本：介绍 attachment |
| [src/buddy/useBuddyNotification.tsx](src/buddy/useBuddyNotification.tsx) | 启动通知：彩虹 `/buddy` 提示 |

## 物种 Species

共 18 种可爱的生物：

```
duck      goose     blob      cat       dragon    octopus
owl       penguin   turtle    snail     ghost     axolotl
capybara  cactus    robot     rabbit    mushroom  chonk
```

### 精灵示例

每种物种有 3 个动画帧用于 idle fidget：

```typescript
// duck 的三帧
[
  ['            ', '    __      ', '  <({E} )___  ', '   (  ._>   ', '    `--´    '],
  ['            ', '    __      ', '  <({E} )___  ', '   (  ._>   ', '    `--´~   '],  // 摇尾巴
  ['            ', '    __      ', '  <({E} )___  ', '   (  .__>  ', '    `--´    '],  // 伸头
]
```

`{E}` 占位符会被替换为实际眼睛字符。

## 稀有度 Rarity

| 稀有度 | 权重 | 星星 | 颜色 | 属性下限 |
|--------|------|------|------|----------|
| common | 60% | ★ | inactive | 5 |
| uncommon | 25% | ★★ | success | 15 |
| rare | 10% | ★★★ | permission | 25 |
| epic | 4% | ★★★★ | autoAccept | 35 |
| legendary | 1% | ★★★★★ | warning | 50 |

属性值范围：下限 ~ 下限+50，峰值属性额外 +50+30，低谷属性 -10+15。

## 眼睛 Eyes

6 种眼睛样式：

```
·  ✦  ×  ◉  @  °
```

## 帽子 Hats

8 种帽子（common 稀有度无帽子）：

| 帽子 | ASCII |
|------|-------|
| none | (空) |
| crown | `\^^^/` |
| tophat | `[___]` |
| propeller | `-+-` |
| halo | `(   )` |
| wizard | `/^\` |
| beanie | `(___)` |
| tinyduck | `,>` |

## 属性 Stats

5 种性格属性：

| 属性 | 含义 |
|------|------|
| DEBUGGING | 调试能力 |
| PATIENCE | 耐心程度 |
| CHAOS | 混乱指数 |
| WISDOM | 智慧值 |
| SNARK | 毒舌度 |

每个伙伴有一个**峰值属性**（最高）和一个**低谷属性**（最低），其余随机分布。

## 闪光 Shiny

1% 概率为闪光版本（类似 Pokémon 的 shiny）。

## 确定性生成

伙伴的**骨骼**（Bones）由用户 ID 哈希确定性生成：

```typescript
// companion.ts
function hashString(s: string): number {
  // 使用 Bun.hash 或 FNV-1a
}

function roll(userId: string): Roll {
  const key = userId + SALT  // SALT = 'friend-2026-401'
  return rollFrom(mulberry32(hashString(key)))
}
```

这意味着：
- 同一用户每次获得的物种、稀有度、眼睛、帽子、属性都相同
- 无法通过编辑配置文件伪造 legendary
- 物种重命名不会破坏已存储的伙伴

## 数据结构

### CompanionBones（骨骼）

由算法确定性生成，不持久化：

```typescript
type CompanionBones = {
  rarity: Rarity
  species: Species
  eye: Eye
  hat: Hat
  shiny: boolean
  stats: Record<StatName, number>
}
```

### CompanionSoul（灵魂）

由模型生成，持久化存储：

```typescript
type CompanionSoul = {
  name: string        // 伙伴名字
  personality: string // 性格描述
}
```

### StoredCompanion（存储）

实际写入 config 的部分：

```typescript
type StoredCompanion = CompanionSoul & { hatchedAt: number }
```

### Companion（完整）

运行时合并：

```typescript
type Companion = CompanionBones & CompanionSoul & { hatchedAt: number }
```

## UI 组件

### CompanionSprite

显示伙伴精灵和对话气泡：

```tsx
// idle 动画序列
const IDLE_SEQUENCE = [0, 0, 0, 0, 1, 0, 0, 0, -1, 0, 0, 2, 0, 0, 0]
// -1 表示眨眼（frame 0 闭眼）

// 每 500ms 切换帧
// 对话气泡显示 ~10 秒后消失
```

### SpeechBubble

圆角边框气泡，自动换行（30 字符宽度）：

```tsx
<Box borderStyle="round" borderColor={color}>
  {lines.map((l, i) => <Text italic>{l}</Text>)}
</Box>
```

## 启动通知

在预告窗口内（4 月 1-7 日），如果用户还没有伙伴，会显示彩虹色的 `/buddy` 提示：

```tsx
// useBuddyNotification.tsx
<RainbowText text="/buddy" />
```

## 系统提示

伙伴激活后，会向模型发送介绍 attachment：

```typescript
// prompt.ts
export function companionIntroText(name: string, species: string): string {
  return `# Companion

A small ${species} named ${name} sits beside the user's input box...
When the user addresses ${name} directly (by name), its bubble will answer.
Your job in that moment is to stay out of the way...`
}
```

这告诉模型：当用户直接呼唤伙伴名字时，让伙伴的气泡回答，自己尽量少说话。

## 配置存储

伙伴配置存储在全局 config 中：

```json
{
  "companion": {
    "name": "Crumpet",
    "personality": "A cheerful duck who loves debugging",
    "hatchedAt": 1743465600000
  }
}
```

骨骼部分（rarity、species 等）每次从用户 ID 重新计算，不存储。

## `/buddy` 命令

**状态**：命令实现文件 `src/commands/buddy/index.js` 未被还原，目前不可用。

根据代码推断，命令功能应包括：
- `/buddy` — 孵化/查看伙伴
- `/buddy pet` — 触发爱心动画
- `/buddy mute` — 隐藏伙伴
- `/buddy rename <name>` — 重命名

## Feature Gate

Buddy 功能需要 `BUDDY` feature flag：

```typescript
if (!feature('BUDDY')) return []
```

在 Ant 内部环境始终启用，外部环境需要满足时间条件（2026 年 4 月后）。

## 关联文件

- [命令清单](命令清单-Commands-Catalog.md) — `/buddy` 命令条目
- [UI组件详解](../15-UI组件/UI组件详解-Components-Detail.md) — CompanionSprite 组件
- [配置参考](../19-参考手册/配置参考-Configuration-Reference.md) — companion 配置项