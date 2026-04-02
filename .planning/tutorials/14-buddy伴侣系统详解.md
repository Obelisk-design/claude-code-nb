# Buddy 伴侣系统详解

**分析日期:** 2026-04-02

## 1. 概述

### 1.1 设计理念 - Tamagotchi 电子宠物

Buddy 系统借鉴了经典的 Tamagotchi（电子宠物）设计理念，为 Claude Code CLI 添加了一个可爱的伴侣精灵。这个小精灵会：

- 坐在用户输入框旁边
- 通过气泡偶尔发表评论
- 与用户互动（可以 pet 抚摸）
- 展示独特的性格和属性

**核心理念：**
- 为冷冰冰的终端界面增添温暖和趣味性
- 通过 Gacha（抽卡）机制带来惊喜感
- 用情感化设计提升用户体验
- 创建用户与 CLI 工具的情感连接

### 1.2 BUDDY Feature Flag

系统通过 `feature('BUDDY')` 控制：

```typescript
// commands.ts
const buddy = feature('BUDDY')
  ? (
      require('./commands/buddy/index.js') as typeof import('./commands/buddy/index.js')
    ).default
  : null
```

**相关配置：**
- `companionMuted` - 是否静音伴侣
- `companion` - 存储伴侣的灵魂数据（名字、性格、孵化时间）

**时间窗口机制：**
```typescript
// useBuddyNotification.tsx
export function isBuddyTeaserWindow(): boolean {
  const d = new Date();
  return d.getFullYear() === 2026 && d.getMonth() === 3 && d.getDate() <= 7;
}
// 活跃期：2026年4月1日-7日（愚人节特别活动）
```

---

## 2. buddy/ 目录逐行分析

### 2.1 buddy/types.ts - 类型定义

**文件路径:** `buddy/types.ts`

#### 稀有度系统

```typescript
export const RARITIES = [
  'common',      // 普通 - 60% 权重
  'uncommon',    // 稀少 - 25% 权重
  'rare',        // 稀有 - 10% 权重
  'epic',        // 史诗 - 4% 权重
  'legendary',   // 传说 - 1% 权重
] as const
```

#### 物种定义（18种）

```typescript
// 使用字符编码避免代码检查工具误报
const c = String.fromCharCode
export const duck = c(0x64,0x75,0x63,0x6b) as 'duck'      // 🦆
export const goose = c(0x67, 0x6f, 0x6f, 0x73, 0x65) as 'goose'  // 🪿
export const blob = c(0x62, 0x6c, 0x6f, 0x62) as 'blob'   // 团状生物
export const cat = c(0x63, 0x61, 0x74) as 'cat'           // 🐱
export const dragon = c(0x64, 0x72, 0x61, 0x67, 0x6f, 0x6e) as 'dragon' // 龙
export const octopus = c(0x6f, 0x63, 0x74, 0x6f, 0x70, 0x75, 0x73) as 'octopus' //章鱼
export const owl = c(0x6f, 0x77, 0x6c) as 'owl'           // 🦉
export const penguin = c(0x70, 0x65, 0x6e, 0x67, 0x75, 0x69, 0x6e) as 'penguin' //企鹅
export const turtle = c(0x74, 0x75, 0x72, 0x74, 0x6c, 0x65) as 'turtle' // 海龟
export const snail = c(0x73, 0x6e, 0x61, 0x69, 0x6c) as 'snail' // 蜗牛
export const ghost = c(0x67, 0x68, 0x6f, 0x73, 0x74) as 'ghost' // 幽灵
export const axolotl = c(0x61, 0x78, 0x6f, 0x6c, 0x6f, 0x74, 0x6c) as 'axolotl' // 蝾螈
export const capybara = c(0x63, 0x61, 0x70, 0x79, 0x62, 0x61, 0x72, 0x61) as 'capybara' // 水豚
export const cactus = c(0x63, 0x61, 0x63, 0x74, 0x75, 0x73) as 'cactus' //仙人掌
export const robot = c(0x72, 0x6f, 0x62, 0x6f, 0x74) as 'robot' // 机器人
export const rabbit = c(0x72, 0x61, 0x62, 0x62, 0x69, 0x74) as 'rabbit' //兔子
export const mushroom = c(0x6d, 0x75, 0x73, 0x68, 0x72, 0x6f, 0x6f, 0x6d) as 'mushroom' //蘑菇
export const chonk = c(0x63, 0x68, 0x6f, 0x6e, 0x6b) as 'chonk' //胖胖生物
```

**设计巧妙之处：**
- 使用 `String.fromCharCode` 编码避免静态分析工具误报（canary 是某个模型的代号）
- 物种名称统一编码，便于批量处理

#### 眼睛样式

```typescript
export const EYES = ['·', '✦', '×', '◉', '@', '°'] as const
// 6种不同的眼睛样式，可替换精灵图中的 {E} 占位符
```

#### 帽子类型

```typescript
export const HATS = [
  'none',       // 无帽子
  'crown',      // 皇冠
  'tophat',     // 高顶帽
  'propeller',  // 螺旋桨帽
  'halo',       //天使光环
  'wizard',     // 巫师帽
  'beanie',     // 无檐便帽
  'tinyduck',   // 小鸭子
] as const
```

#### 属性系统

```typescript
export const STAT_NAMES = [
  'DEBUGGING',  // 调试能力
  'PATIENCE',   // 耐心值
  'CHAOS',      // 混乱度
  'WISDOM',     // 智慧值
  'SNARK',      // 讽刺值
] as const
// 每个属性值范围 1-100，根据稀有度有不同的基础值
```

#### 数据结构

```typescript
// 确定性部分 - 从 userId 哈希派生
export type CompanionBones = {
  rarity: Rarity
  species: Species
  eye: Eye
  hat: Hat
  shiny: boolean      // 闪光属性（1%概率）
  stats: Record<StatName, number>
}

// 模型生成的灵魂 - 存储在配置中
export type CompanionSoul = {
  name: string         // 名字
  personality: string  // 性格描述
}

// 完整伴侣类型
export type Companion = CompanionBones &
  CompanionSoul & {
    hatchedAt: number  // 孵化时间戳
  }
```

#### 稀有度权重与颜色

```typescript
export const RARITY_WEIGHTS = {
  common: 60,
  uncommon: 25,
  rare: 10,
  epic: 4,
  legendary: 1,
} as const

export const RARITY_STARS = {
  common: '★',
  uncommon: '★★',
  rare: '★★★',
  epic: '★★★★',
  legendary: '★★★★★',
} as const

export const RARITY_COLORS = {
  common: 'inactive',      // 灰色
  uncommon: 'success',     // 绿色
  rare: 'permission',      // 蓝色
  epic: 'autoAccept',      // 黄色
  legendary: 'warning',    // 红色
} as const
```

---

### 2.2 buddy/companion.ts - 核心逻辑

**文件路径:** `buddy/companion.ts`

#### Mulberry32 PRNG - 确定性随机

```typescript
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
```

**设计原理：**
- 使用种子随机数确保同一用户每次获得相同的伴侣
- 防止用户通过修改配置获得稀有精灵

#### 字符串哈希

```typescript
function hashString(s: string): number {
  if (typeof Bun !== 'undefined') {
    // Bun 环境使用内置哈希
    return Number(BigInt(Bun.hash(s)) & 0xffffffffn)
  }
  // FNV-1a 哈希算法
  let h = 2166136261
  for (let i = 0; i < s.length; i++) {
    h ^= s.charCodeAt(i)
    h = Math.imul(h, 16777619)
  }
  return h >>> 0
}
```

#### 稀有度抽取

```typescript
function rollRarity(rng: () => number): Rarity {
  const total = Object.values(RARITY_WEIGHTS).reduce((a, b) => a + b, 0)
  let roll = rng() * total
  for (const rarity of RARITIES) {
    roll -= RARITY_WEIGHTS[rarity]
    if (roll < 0) return rarity
  }
  return 'common'
}
// 概率分布：60% 普通, 25% 稀少, 10% 稀有, 4% 史诗, 1% 传说
```

#### 属性生成

```typescript
const RARITY_FLOOR: Record<Rarity, number> = {
  common: 5,
  uncommon: 15,
  rare: 25,
  epic: 35,
  legendary: 50,
}

function rollStats(
  rng: () => number,
  rarity: Rarity,
): Record<StatName, number> {
  const floor = RARITY_FLOOR[rarity]
  const peak = pick(rng, STAT_NAMES)   // 一个巅峰属性
  let dump = pick(rng, STAT_NAMES)      // 一个低谷属性
  while (dump === peak) dump = pick(rng, STAT_NAMES)

  const stats = {} as Record<StatName, number>
  for (const name of STAT_NAMES) {
    if (name === peak) {
      stats[name] = Math.min(100, floor + 50 + Math.floor(rng() * 30))
    } else if (name === dump) {
      stats[name] = Math.max(1, floor - 10 + Math.floor(rng() * 15))
    } else {
      stats[name] = floor + Math.floor(rng() * 40)
    }
  }
  return stats
}
```

**属性生成规则：**
- 每个伴侣有一个巅峰属性（高值）
- 一个低谷属性（低值）
- 其他属性随机分布在基础值附近
- 稀有度决定基础值（传说精灵所有属性至少50）

#### Roll 函数

```typescript
const SALT = 'friend-2026-401'

export type Roll = {
  bones: CompanionBones
  inspirationSeed: number
}

function rollFrom(rng: () => number): Roll {
  const rarity = rollRarity(rng)
  const bones: CompanionBones = {
    rarity,
    species: pick(rng, SPECIES),
    eye: pick(rng, EYES),
    hat: rarity === 'common' ? 'none' : pick(rng, HATS), // 普通无帽子
    shiny: rng() < 0.01,  // 1% 闪光概率
    stats: rollStats(rng, rarity),
  }
  return { bones, inspirationSeed: Math.floor(rng() * 1e9) }
}
```

#### 缓存机制

```typescript
let rollCache: { key: string; value: Roll } | undefined

export function roll(userId: string): Roll {
  const key = userId + SALT
  if (rollCache?.key === key) return rollCache.value
  const value = rollFrom(mulberry32(hashString(key)))
  rollCache = { key, value }
  return value
}
```

**缓存目的：**
- 避免频繁调用时重复计算
- 三个热点路径：500ms 精灵动画、每次按键、每轮观察者

#### 获取伴侣

```typescript
export function companionUserId(): string {
  const config = getGlobalConfig()
  return config.oauthAccount?.accountUuid ?? config.userID ?? 'anon'
}

export function getCompanion(): Companion | undefined {
  const stored = getGlobalConfig().companion
  if (!stored) return undefined
  const { bones } = roll(companionUserId())
  // bones 后置合并，覆盖旧格式配置中的过时字段
  return { ...stored, ...bones }
}
```

**安全设计：**
- Bones 不持久化，每次从 userId 重新生成
- 防止用户通过编辑配置伪造稀有度
- 物种重命名不影响已存储的伴侣

---

### 2.3 buddy/sprites.ts - 精灵图定义

**文件路径:** `buddy/sprites.ts`

#### 精灵图结构

```typescript
// 每个精灵 5 行高，12 列宽
// {E} 占位符会被眼睛样式替换
// 帽子槽在第 0 行（部分动画帧使用）
const BODIES: Record<Species, string[][]> = {
  [duck]: [
    [
      '            ',       //帽子槽（空）
      '    __      ',       // 身体第1行
      '  <({E} )___  ',     // {E} 替换为眼睛
      '   (  ._>   ',       // 身体第3行
      '    `--´    ',       // 身体第4行
    ],
    // ... 多个动画帧
  ],
  // ... 其他物种
}
```

#### Duck 精灵详解

```typescript
[duck]: [
  // Frame 0 - 静止态
  [
    '            ',
    '    __      ',
    '  <({E} )___  ',  // 眼睛位置
    '   (  ._>   ',    //嘴巴
    '    `--´    ',     //尾巴
  ],
  // Frame 1 - 小晃动
  [
    '            ',
    '    __      ',
    '  <({E} )___  ',
    '   (  ._>   ',
    '    `--´~   ',     //尾巴加波浪
  ],
  // Frame 2 - 大晃动
  [
    '            ',
    '    __      ',
    '  <({E} )___  ',
    '   (  .__>  ',     // 嘴巴变化
    '    `--´    ',
  ],
]
```

#### Cat 精灵

```typescript
[cat]: [
  [
    '            ',
    '   /\\_/\\    ',     //耳朵
    '  ( {E}   {E})  ',   //双眼
    '  (  ω  )   ',      // 嘴巴 (ω表情)
    '  (")_(")   ',      // 脚
  ],
  // ...
]
```

#### Dragon 精灵

```typescript
[dragon]: [
  [
    '            ',
    '  /^\\  /^\\  ',    // 双角
    ' <  {E}  {E}  > ',   //双眼
    ' (   ~~   ) ',      // 嘴巴（喷气效果）
    '  `-vvvv-´  ',       // 翅膀
  ],
  // Frame 2 有烟雾效果（第0行）
  [
    '   ~    ~   ',      // 烟雾
    '  /^\\  /^\\  ',
    ' <  {E}  {E}  > ',
    ' (   ~~   ) ',
    '  `-vvvv-´  ',
  ],
]
```

#### Ghost 精灵

```typescript
[ghost]: [
  [
    '            ',
    '   .----.   ',
    '  / {E}  {E} \\  ',   // 双眼
    '  |      |  ',
    '  ~`~``~`~  ',       // 幽灵底部波浪
  ],
  // ...
]
```

#### 帽子渲染

```typescript
const HAT_LINES: Record<Hat, string> = {
  none: '',
  crown: '   \\^^^/    ',
  tophat: '   [___]    ',
  propeller: '    -+-     ',
  halo: '   (   )    ',
  wizard: '    /^\\     ',
  beanie: '   (___)    ',
  tinyduck: '    ,>      ',
}
```

#### 渲染函数

```typescript
export function renderSprite(bones: CompanionBones, frame = 0): string[] {
  const frames = BODIES[bones.species]
  const body = frames[frame % frames.length]!.map(line =>
    line.replaceAll('{E}', bones.eye),  // 替换眼睛占位符
  )
  const lines = [...body]
  
  // 帽子逻辑
  if (bones.hat !== 'none' && !lines[0]!.trim()) {
    lines[0] = HAT_LINES[bones.hat]
  }
  
  // 如果所有帧的第0行都空，删除帽子槽行
  if (!lines[0]!.trim() && frames.every(f => !f[0]!.trim())) {
    lines.shift()
  }
  
  return lines
}
```

#### 简化面部渲染

```typescript
export function renderFace(bones: CompanionBones): string {
  const eye: Eye = bones.eye
  switch (bones.species) {
    case duck:
    case goose:
      return `(${eye}>`
    case blob:
      return `(${eye}${eye})`
    case cat:
      return `=${eye}ω${eye}=`
    case dragon:
      return `<${eye}~${eye}>`
    case octopus:
      return `~(${eye}${eye})~`
    case owl:
      return `(${eye})(${eye})`
    case penguin:
      return `(${eye}>)`
    case turtle:
      return `[${eye}_${eye}]`
    case snail:
      return `${eye}(@)`
    case ghost:
      return `/${eye}${eye}\\`
    case axolotl:
      return `}${eye}.${eye}{`
    case capybara:
      return `(${eye}oo${eye})`
    case cactus:
      return `|${eye}  ${eye}|`
    case robot:
      return `[${eye}${eye}]`
    case rabbit:
      return `(${eye}..${eye})`
    case mushroom:
      return `|${eye}  ${eye}|`
    case chonk:
      return `(${eye}.${eye})`
  }
}
```

---

### 2.4 buddy/CompanionSprite.tsx - React 组件

**文件路径:** `buddy/CompanionSprite.tsx`

#### 常量配置

```typescript
const TICK_MS = 500                // 动画帧间隔（500ms）
const BUBBLE_SHOW = 20             // 气泡显示时长（20ticks ≈10s）
const FADE_WINDOW = 6              // 气泡淡出窗口（最后3s）
const PET_BURST_MS = 2500          // pet 后心形动画时长

// Idle 动画序列
const IDLE_SEQUENCE = [0, 0, 0, 0, 1, 0, 0, 0, -1, 0, 0, 2, 0, 0, 0]
// -1 表示眨眼（frame 0 替换眼睛为 '-'）
```

#### Pet 心形动画

```typescript
const H = figures.heart
const PET_HEARTS = [
  `   ${H}    ${H}   `,     // 第1帧
  `  ${H}  ${H}   ${H}  `,   // 第2帧
  ` ${H}   ${H}  ${H}   `,   // 第3帧
  `${H}  ${H}      ${H} `,   // 第4帧
  '·    ·   ·  ',             // 第5帧（消散）
]
```

#### SpeechBubble 组件

```typescript
function SpeechBubble({
  text,
  color,
  fading,
  tail,
}: {
  text: string
  color: keyof Theme
  fading: boolean
  tail: 'down' | 'right'
}): React.ReactNode {
  const lines = wrap(text, 30)   // 文本自动换行
  const borderColor = fading ? 'inactive' : color
  
  return (
    <Box
      flexDirection="column"
      borderStyle="round"
      borderColor={borderColor}
      paddingX={1}
      width={34}
    >
      {lines.map((l, i) => (
        <Text key={i} italic dimColor={!fading} color={fading ? 'inactive' : undefined}>
          {l}
        </Text>
      ))}
    </Box>
  )
}
```

#### CompanionSprite 主组件

```typescript
export function CompanionSprite(): React.ReactNode {
  const reaction = useAppState(s => s.companionReaction)
  const petAt = useAppState(s => s.companionPetAt)
  const focused = useAppState(s => s.footerSelection === 'companion')
  const setAppState = useSetAppState()
  const { columns } = useTerminalSize()
  const [tick, setTick] = useState(0)
  const lastSpokeTick = useRef(0)
  
  // 动画定时器
  useEffect(() => {
    const timer = setInterval(
      setT => setT((t: number) => t + 1),
      TICK_MS,
      setTick,
    )
    return () => clearInterval(timer)
  }, [])
  
  // 气泡自动消失
  useEffect(() => {
    if (!reaction) return
    lastSpokeTick.current = tick
    const timer = setTimeout(
      setA => setA((prev: AppState) =>
        prev.companionReaction === undefined ? prev : {
          ...prev,
          companionReaction: undefined
        }),
      BUBBLE_SHOW * TICK_MS,
      setAppState,
    )
    return () => clearTimeout(timer)
  }, [reaction, setAppState])
  
  // ...
}
```

#### 动画帧选择

```typescript
let spriteFrame: number
let blink = false

if (reaction || petting) {
  // 激动状态：快速循环所有晃动帧
  spriteFrame = tick % frameCount
} else {
  // Idle 状态：按序列播放
  const step = IDLE_SEQUENCE[tick % IDLE_SEQUENCE.length]!
  if (step === -1) {
    spriteFrame = 0
    blink = true           // 眨眼
  } else {
    spriteFrame = step % frameCount
  }
}

// 渲染时处理眨眼
const body = renderSprite(companion, spriteFrame).map(line =>
  blink ? line.replaceAll(companion.eye, '-') : line
)
```

#### 精灵渲染

```typescript
const spriteColumn = (
  <Box flexDirection="column" flexShrink={0} alignItems="center" width={colWidth}>
    {sprite.map((line, i) => (
      <Text key={i} color={i === 0 && heartFrame ? 'autoAccept' : color}>
        {line}
      </Text>
    ))}
    <Text italic bold={focused} dimColor={!focused} color={focused ? color : undefined} inverse={focused}>
      {focused ? ` ${companion.name} ` : companion.name}
    </Text>
  </Box>
)
```

#### 全屏模式处理

```typescript
// 全屏模式：气泡浮动在滚动区上方
if (isFullscreenActive()) {
  return <Box paddingX={1}>{spriteColumn}</Box>
}

// 非全屏：气泡内联显示
return (
  <Box flexDirection="row" alignItems="flex-end" paddingX={1} flexShrink={0}>
    <SpeechBubble text={reaction} color={color} fading={fading} tail="right" />
    {spriteColumn}
  </Box>
)
```

#### 空间预留计算

```typescript
export const MIN_COLS_FOR_FULL_SPRITE = 100

export function companionReservedColumns(
  terminalColumns: number,
  speaking: boolean,
): number {
  if (!feature('BUDDY')) return 0
  const companion = getCompanion()
  if (!companion || getGlobalConfig().companionMuted) return 0
  if (terminalColumns < MIN_COLS_FOR_FULL_SPRITE) return 0
  
  const nameWidth = stringWidth(companion.name)
  const bubble = speaking && !isFullscreenActive() ? BUBBLE_WIDTH : 0
  return spriteColWidth(nameWidth) + SPRITE_PADDING_X + bubble
}
```

---

### 2.5 buddy/prompt.ts - 提示词

**文件路径:** `buddy/prompt.ts`

#### 伴侣介绍文本

```typescript
export function companionIntroText(name: string, species: string): string {
  return `# Companion

A small ${species} named ${name} sits beside the user's input box and occasionally comments in a speech bubble. You're not ${name} — it's a separate watcher.

When the user addresses ${name} directly (by name), its bubble will answer. Your job in that moment is to stay out of the way: respond in ONE line or less, or just answer any part of the message meant for you. Don't explain that you're not ${name} — they know. Don't narrate what ${name} might say — the bubble handles that.`
}
```

#### 介绍附件生成

```typescript
export function getCompanionIntroAttachment(
  messages: Message[] | undefined,
): Attachment[] {
  if (!feature('BUDDY')) return []
  const companion = getCompanion()
  if (!companion || getGlobalConfig().companionMuted) return []

  // 检查是否已经介绍过这个伴侣
  for (const msg of messages ?? []) {
    if (msg.type !== 'attachment') continue
    if (msg.attachment.type !== 'companion_intro') continue
    if (msg.attachment.name === companion.name) return []
  }

  return [
    {
      type: 'companion_intro',
      name: companion.name,
      species: companion.species,
    },
  ]
}
```

---

### 2.6 buddy/useBuddyNotification.tsx - 通知 Hook

**文件路径:** `buddy/useBuddyNotification.tsx`

#### Rainbow 文本组件

```typescript
function RainbowText({ text }: { text: string }): React.ReactNode {
  return (
    <>
      {[...text].map((ch, i) => (
        <Text key={i} color={getRainbowColor(i)}>
          {ch}
        </Text>
      ))}
    </>
  )
}
// 为 /buddy 命令显示彩虹效果
```

#### 通知 Hook

```typescript
export function useBuddyNotification(): void {
  const { addNotification, removeNotification } = useNotifications()

  useEffect(() => {
    if (!feature('BUDDY')) return
    const config = getGlobalConfig()
    
    // 条件：无伴侣 + 处于预告期
    if (config.companion || !isBuddyTeaserWindow()) return
    
    // 显示彩虹 /buddy 提示
    addNotification({
      key: 'buddy-teaser',
      jsx: <RainbowText text="/buddy" />,
      priority: 'immediate',
      timeoutMs: 15000,
    })
    
    return () => removeNotification('buddy-teaser')
  }, [addNotification, removeNotification])
}
```

#### Trigger 检测

```typescript
export function findBuddyTriggerPositions(text: string): Array<{
  start: number
  end: number
}> {
  if (!feature('BUDDY')) return []
  const triggers: Array<{ start: number; end: number }> = []
  const re = /\/buddy\b/g
  let m: RegExpExecArray | null
  
  while ((m = re.exec(text)) !== null) {
    triggers.push({
      start: m.index,
      end: m.index + m[0].length,
    })
  }
  return triggers
}
```

---

## 3. Gacha 系统分析

### 3.1 精灵抽取机制

**核心流程：**

```
用户ID → hashString → mulberry32(seed) → RNG序列
         ↓
         rollFrom(rng)
         ↓
         ├── rollRarity() → 稀有度
         ├── pick(rng, SPECIES) → 物种
         ├── pick(rng, EYES) → 眼睛
         ├── pick(rng, HATS) → 帽子（普通除外）
         ├── rng() < 0.01 → 闪光
         └── rollStats() → 属性值
```

**确定性设计：**
- 同一用户永远获得相同的精灵
- SALT（`friend-2026-401`）可调整全局分配
- 用户无法通过编辑配置改变精灵属性

### 3.2 精灵种类和稀有度

**18种物种分布：**
| 类型 | 数量 | 例子 |
|------|------|------|
| 动物 | 10 | duck, cat, rabbit, owl, penguin, turtle, snail, axolotl, capybara, ghost |
| 神兽 | 1 | dragon |
| 植物 | 2 | cactus, mushroom |
| 机械 | 1 | robot |
| 抽象 | 4 | blob, goose, chonk, octopus |

**稀有度概率：**
| 稀有度 | 权重 | 概率 | 基础属性 |帽子 |
|--------|------|------|----------|------|
| common | 60 | 60% | 5 | 无 |
| uncommon | 25 | 25% | 15 | 有 |
| rare | 10 | 10% | 25 | 有 |
| epic | 4 | 4% | 35 | 有 |
| legendary | 1 | 1% | 50 | 有 |

**特殊属性：**
- Shiny（闪光）：1%概率，可能带有特效
- 属性分布：1个巅峰 + 1个低谷 + 3个随机

---

## 4. Buddy 交互机制

### 4.1 喂食/互动（Pet）

**触发方式：**
```bash
/buddy pet
```

**交互效果：**
```typescript
// CompanionSprite.tsx
const petAge = petAt ? tick - petStartTick : Infinity
const petting = petAge * TICK_MS < PET_BURST_MS  // 2500ms内显示心形
```

**心形动画：**
- 5帧动画，每帧500ms
- 心形从下往上飘起
- 最后消散成点状

### 4.2 情绪状态

**动画状态：**

| 状态 | 动画模式 | 触发条件 |
|------|----------|----------|
| Idle | 序列播放 | 正常状态 |
| Blink | 眨眼 | 序列中 -1 |
| Fidget | 晃动 | 序列中 1,2 |
| Excited | 快速循环 | reaction 或 petting |

**Idle 序列分析：**
```typescript
const IDLE_SEQUENCE = [0, 0, 0, 0, 1, 0, 0, 0, -1, 0, 0, 2, 0, 0, 0]
//                       ↑  ↑  ↑  ↑  ↑  ↑  ↑  ↑   ↑  ↑  ↑  ↑  ↑  ↑  ↑
//                       静  静  静  静  晃  静  静  静  眨  静  静  大  静  静  静
// 15帧循环 = 7.5秒周期
```

### 4.3 成长系统

**当前实现：**
- `hatchedAt` 记录孵化时间
- 名字和性格由模型生成
- 属性由稀有度决定基础值

**潜在扩展：**
- 根据使用时长解锁新帽子
- 互动增加属性值
- 进化系统（形态变化）

---

## 5. /buddy 命令实现

### 5.1 命令注册

```typescript
// commands.ts
const buddy = feature('BUDDY')
  ? (
      require('./commands/buddy/index.js') as typeof import('./commands/buddy/index.js')
    ).default
  : null

// 命令列表
...(buddy ? [buddy] : []),
```

### 5.2 命令入口

**入口路径:** `commands/buddy/index.js`

**功能推测：**
- `/buddy` - 孵化新伴侣
- `/buddy pet` - 抚摸伴侣
- `/buddy rename` - 重命名
- `/buddy stats` - 查看属性
- `/buddy mute` - 静音

### 5.3 Trigger 检测

```typescript
// PromptInput.tsx
const buddyTriggers = useMemo(
  () => findBuddyTriggerPositions(displayedValue),
  [displayedValue]
)

// 彩虹高亮
for (const trigger of buddyTriggers) {
  for (let i = trigger.start; i < trigger.end; i++) {
    highlights.push({
      start: i,
      end: i + 1,
      color: getRainbowColor(i - trigger.start),
      type: 'default',
      priority: 10,
    })
  }
}
```

### 5.4 Footer 集成

```typescript
// PromptInput.tsx
const companionFooterVisible = !!_companion && !companionMuted

const footerItems = useMemo(() => [
  tasksFooterVisible && 'tasks',
  tmuxFooterVisible && 'tmux',
  bagelFooterVisible && 'bagel',
  teamsFooterVisible && 'teams',
  bridgeFooterVisible && 'bridge',
  companionFooterVisible && 'companion',  //伴侣 footer
].filter(Boolean) as FooterItem[], [/* deps */])

// 键盘导航
case 'companion':
  if (feature('BUDDY')) {
    selectFooterItem(null);
    void onSubmit('/buddy');
  }
  break;
```

---

## 6. 从零实现指南

### 6.1 如何设计 CLI 伴侣系统

**核心设计要点：**

1. **确定性随机**
   - 使用种子随机数
   - 用户ID作为种子源
   - 添加 SALT 防止枚举

2. **数据分离**
   - 确定性属性（外观）每次重新生成
   - 个性化属性（名字）持久存储
   - 防止配置篡改

3. **渲染优化**
   - 缓存 roll 结果
   - 按需渲染（仅在可见时）
   - 动画帧预计算

### 6.2 状态机设计

**伴侣状态机：**

```
┌─────────┐     reaction     ┌───────────┐
│  IDLE   │────────────────→│  SPEAKING │
└─────────┘                  └───────────┘
     ↑                            │
     │       timeout               │
     │    (BUBBLE_SHOW ticks)      │
     └─────────────────────────────┘
     
     │        pet command          │
     ↓
┌─────────┐    2.5s timeout    ┌─────────┐
│PETTING │────────────────→    │ IDLE    │
└─────────┘                     └─────────┘
```

**动画状态机：**

```
IDLE_SEQUENCE: [0,0,0,0,1,0,0,0,-1,0,0,2,0,0,0]

Frame 0: 静止 ──┐
Frame 1: 小晃动 ─┤
Frame 2: 大晃动 ─┤
-1: 眨眼       ─┤
               └─→ 15帧循环（7.5秒）
```

### 6.3 终端精灵渲染

**ASCII 艺术技巧：**

1. **占位符替换**
```typescript
// 使用 {E} 作为眼睛占位符
'  <({E} )___  '.replaceAll('{E}', eye)
```

2. **多帧动画**
```typescript
// 每个物种3帧动画
const frames = BODIES[species]
const currentFrame = frames[tick % frames.length]
```

3. **帽子叠加**
```typescript
// 第0行预留帽子槽
if (hat !== 'none' && !lines[0].trim()) {
  lines[0] = HAT_LINES[hat]
}
```

4. **响应式设计**
```typescript
// 宽终端：完整精灵
// 窄终端：简化面部
if (columns < MIN_COLS_FOR_FULL_SPRITE) {
  return renderFace(companion)  //一行面部
}
```

### 6.4 实现步骤

**Phase 1: 类型系统**
```typescript
// 1. 定义稀有度、物种、眼睛、帽子
// 2. 定义 CompanionBones 和 CompanionSoul
// 3. 定义权重和颜色映射
```

**Phase 2: 核心逻辑**
```typescript
// 1. 实现 mulberry32 PRNG
// 2. 实现 hashString
// 3. 实现 rollRarity、rollStats
// 4. 实现 roll 和 getCompanion
```

**Phase 3: 精灵渲染**
```typescript
// 1. 定义 BODIES 精灵图
// 2. 定义 HAT_LINES帽子
// 3. 实现 renderSprite
// 4. 实现 renderFace
```

**Phase 4: React 组件**
```typescript
// 1. 创建 SpeechBubble组件
// 2. 创建 CompanionSprite组件
// 3. 实现动画定时器
// 4. 实现状态管理
```

**Phase 5: 命令集成**
```typescript
// 1. 创建 /buddy 命令
// 2. 注册 feature flag
// 3. 集成 PromptInput
// 4. 集成 REPL 界面
```

---

## 附录

### A. 文件路径索引

| 文件 | 路径 | 功能 |
|------|------|------|
| types.ts | `buddy/types.ts` | 类型定义 |
| companion.ts | `buddy/companion.ts` | 核心逻辑 |
| sprites.ts | `buddy/sprites.ts` | 精灵图 |
| CompanionSprite.tsx | `buddy/CompanionSprite.tsx` | React组件 |
| prompt.ts | `buddy/prompt.ts` | 提示词 |
| useBuddyNotification.tsx | `buddy/useBuddyNotification.tsx` | 通知Hook |

### B. 状态存储位置

| 数据 | 存储位置 | 类型 |
|------|----------|------|
| companion | `config.companion` | StoredCompanion |
| companionReaction | `AppState.companionReaction` | string |
| companionPetAt | `AppState.companionPetAt` | number |
| companionMuted | `config.companionMuted` | boolean |

### C. 物种完整列表

```
duck        -鸭子（经典）
goose       - 鹅
blob        - 团状生物
cat         -猫
dragon      - 龙
octopus     -章鱼
owl         - 猫头鹰
penguin     -企鹅
turtle      - 海龟
snail       - 蜗牛
ghost       - 幽灵
axolotl     - 蝾螈
capybara    - 水豚
cactus      -仙人掌
robot       - 机器人
rabbit      -兔子
mushroom    - 蘑菇
chonk       -胖胖生物
```

---

*Buddy系统分析: 2026-04-02*