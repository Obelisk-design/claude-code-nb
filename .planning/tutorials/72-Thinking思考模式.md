# Thinking思考模式详解

**分析日期:** 2026-04-02

## 1. Thinking概述

Thinking（扩展思考）是Claude Code CLI的核心特性，允许Claude模型在响应前进行深度思考。这个系统涉及多个层面：

- **API层**：向Anthropic API发送thinking配置参数
- **流处理层**：处理thinking和redacted_thinking内容块
- **UI层**：渲染思考内容，支持折叠/展开
- **配置层**：用户设置和环境变量控制

### 核心类型定义

```typescript
// utils/thinking.ts:10-13
export type ThinkingConfig =
  | { type: 'adaptive' }      // 自适应思考（无预算限制）
  | { type: 'enabled'; budgetTokens: number }  // 固定预算思考
  | { type: 'disabled' }      // 禁用思考
```

三种模式说明：
- **adaptive**：模型自行决定思考深度，无token预算限制（推荐用于Claude 4.6+）
- **enabled**：设置固定的thinking token预算
- **disabled**：完全禁用扩展思考

---

## 2. utils/thinking.ts 逐行分析

### 2.1 导入和依赖

```typescript
// utils/thinking.ts:1-8
import type { Theme } from './theme.js'
import { feature } from 'bun:bundle'  // 构建时特性开关
import { getFeatureValue_CACHED_MAY_BE_STALE } from '../services/analytics/growthbook.js'
import { getCanonicalName } from './model/model.js'
import { get3PModelCapabilityOverride } from './model/modelSupportOverrides.js'
import { getAPIProvider } from './model/providers.js'
import { getSettingsWithErrors } from './settings/settings.js'
```

关键依赖：
- `bun:bundle`：Bun的构建时特性开关系统
- `growthbook.js`：A/B测试和特性发布平台
- `model/model.js`：模型名称规范化
- `modelSupportOverrides.ts`：第三方模型能力覆盖

### 2.2 Ultrathink功能（内部特性）

```typescript
// utils/thinking.ts:19-24
export function isUltrathinkEnabled(): boolean {
  if (!feature('ULTRATHINK')) {
    return false  // 构建时开关关闭则直接返回false
  }
  return getFeatureValue_CACHED_MAY_BE_STALE('tengu_turtle_carbon', true)
}
```

Ultrathink是内部特性，用于触发超深度思考：
- **双层开关**：构建时feature开关 + 运行时GrowthBook开关
- **特性标识**：`tengu_turtle_carbon`

```typescript
// utils/thinking.ts:29-31
export function hasUltrathinkKeyword(text: string): boolean {
  return /\bultrathink\b/i.test(text)  // 正则匹配关键词
}
```

用户输入中检测"ultrathink"关键词，触发超深度思考模式。

```typescript
// utils/thinking.ts:36-58
export function findThinkingTriggerPositions(text: string): Array<{
  word: string
  start: number
  end: number
}> {
  const positions: Array<{ word: string; start: number; end: number }> = []
  // 关键注释：每次调用使用新的/g正则，避免状态泄漏
  const matches = text.matchAll(/\bultrathink\b/gi)

  for (const match of matches) {
    if (match.index !== undefined) {
      positions.push({
        word: match[0],
        start: match.index,
        end: match.index + match[0].length,
      })
    }
  }

  return positions
}
```

**设计要点**：注释解释了为什么每次创建新的正则表达式：
> Fresh /g literal each call — String.prototype.matchAll copies lastIndex from the source regex, so a shared instance would leak state from hasUltrathinkKeyword's .test() into this call on the next render.

这是正则表达式状态管理的经典陷阱，共享的/g正则会保留lastIndex状态。

### 2.3 彩虹色渲染系统

```typescript
// utils/thinking.ts:60-78
const RAINBOW_COLORS: Array<keyof Theme> = [
  'rainbow_red', 'rainbow_orange', 'rainbow_yellow',
  'rainbow_green', 'rainbow_blue', 'rainbow_indigo', 'rainbow_violet',
]

const RAINBOW_SHIMMER_COLORS: Array<keyof Theme> = [
  'rainbow_red_shimmer', 'rainbow_orange_shimmer', 'rainbow_yellow_shimmer',
  'rainbow_green_shimmer', 'rainbow_blue_shimmer', 'rainbow_indigo_shimmer', 'rainbow_violet_shimmer',
]

export function getRainbowColor(charIndex: number, shimmer: boolean = false): keyof Theme {
  const colors = shimmer ? RAINBOW_SHIMMER_COLORS : RAINBOW_COLORS
  return colors[charIndex % colors.length]!
}
```

彩虹色用于：
- Ultrathink关键词的高亮显示
- Buddy伴侣系统的彩虹文字效果

### 2.4 模型思考能力检测

```typescript
// utils/thinking.ts:90-110
export function modelSupportsThinking(model: string): boolean {
  // 1. 先检查第三方模型能力覆盖
  const supported3P = get3PModelCapabilityOverride(model, 'thinking')
  if (supported3P !== undefined) {
    return supported3P
  }

  // 2. 内部用户检查Ant模型
  if (process.env.USER_TYPE === 'ant') {
    if (resolveAntModel(model.toLowerCase())) {
      return true
    }
  }

  // 3. 关键注释：修改思考支持需要通知模型启动DRI和研究团队
  // IMPORTANT: Do not change thinking support without notifying the model
  // launch DRI and research. This can greatly affect model quality and bashing.

  const canonical = getCanonicalName(model)
  const provider = getAPIProvider()

  // 4. 1P和Foundry：所有Claude 4+模型（包括Haiku 4.5）
  if (provider === 'foundry' || provider === 'firstParty') {
    return !canonical.includes('claude-3-')  // 排除Claude 3系列
  }

  // 5. 3P（Bedrock/Vertex）：仅Opus 4+和Sonnet 4+
  return canonical.includes('sonnet-4') || canonical.includes('opus-4')
}
```

**重要决策逻辑**：
- 第一方API和Foundry代理：所有Claude 4+支持
- 第三方API（Bedrock/Vertex）：仅高端模型支持
- Claude 3系列不支持扩展思考

### 2.5 自适应思考能力检测

```typescript
// utils/thinking.ts:113-144
export function modelSupportsAdaptiveThinking(model: string): boolean {
  // 1. 检查第三方覆盖
  const supported3P = get3PModelCapabilityOverride(model, 'adaptive_thinking')
  if (supported3P !== undefined) {
    return supported3P
  }

  const canonical = getCanonicalName(model)

  // 2. 仅Claude 4.6变体支持自适应思考
  if (canonical.includes('opus-4-6') || canonical.includes('sonnet-4-6')) {
    return true
  }

  // 3. 排除其他已知旧模型（白名单优先）
  if (
    canonical.includes('opus') ||
    canonical.includes('sonnet') ||
    canonical.includes('haiku')
  ) {
    return false
  }

  // 4. 关键注释：新模型默认启用自适应思考
  // IMPORTANT: Do not change adaptive thinking support without notifying...
  // Newer models (4.6+) are all trained on adaptive thinking and MUST have it
  // enabled for model testing.

  // 5. 默认策略：1P和Foundry返回true（未知模型可能是新版本）
  const provider = getAPIProvider()
  return provider === 'firstParty' || provider === 'foundry'
}
```

**自适应思考vs固定预算思考**：
- 自适应思考：模型自行决定深度，无预算限制
- 固定预算：预设thinking token上限

### 2.6 默认启用检测

```typescript
// utils/thinking.ts:146-162
export function shouldEnableThinkingByDefault(): boolean {
  // 1. 环境变量优先
  if (process.env.MAX_THINKING_TOKENS) {
    return parseInt(process.env.MAX_THINKING_TOKENS, 10) > 0
  }

  // 2. 用户设置检查
  const { settings } = getSettingsWithErrors()
  if (settings.alwaysThinkingEnabled === false) {
    return false  // 用户显式禁用
  }

  // 3. 关键注释：默认启用策略
  // IMPORTANT: Do not change default thinking enabled value without notifying
  // the model launch DRI and research. This can greatly affect model quality.

  // Enable thinking by default unless explicitly disabled.
  return true
}
```

**默认行为**：思考默认启用，除非用户显式设置`alwaysThinkingEnabled: false`。

---

## 3. 思考块处理

### 3.1 思考规则（query.ts）

```typescript
// query.ts:151-163
/**
 * The rules of thinking are lengthy and fortuitous...
 *
 * The rules follow:
 * 1. A message that contains a thinking or redacted_thinking block must be part of a query whose max_thinking_length > 0
 * 2. A thinking block may not be the last message in a block
 * 3. Thinking blocks must be preserved for the duration of an assistant trajectory (a single turn, or if that turn includes a tool_use block then also its subsequent tool_result and the following assistant message)
 *
 * Heed these rules well, young wizard...
 */
```

**三条铁律**：
1. 包含thinking块的消息必须有thinking配置
2. thinking块不能是消息块的最后一个
3. thinking块必须在整个assistant轨迹期间保留

### 3.2 流式处理初始化（services/api/claude.ts）

```typescript
// services/api/claude.ts:2030-2038
case 'thinking':
  contentBlocks[part.index] = {
    ...part.content_block,
    thinking: '',  // 初始化为空字符串
    signature: '',  // 确保signature字段存在
  }
  break
```

思考块初始化：
- `thinking`字段初始化为空字符串，通过delta累积
- `signature`字段初始化，即使后续没有signature_delta

### 3.3 思考增量处理

```typescript
// services/api/claude.ts:2148-2161
case 'thinking_delta':
  if (contentBlock.type !== 'thinking') {
    logEvent('tengu_streaming_error', {
      error_type: 'content_block_type_mismatch_thinking_delta',
      expected_type: 'thinking',
      actual_type: contentBlock.type,
    })
    throw new Error('Content block is not a thinking block')
  }
  contentBlock.thinking += delta.thinking  // 累积思考文本
  break
```

```typescript
// services/api/claude.ts:2127-2147
case 'signature_delta':
  if (contentBlock.type !== 'thinking') {
    throw new Error('Content block is not a thinking block')
  }
  contentBlock.signature = delta.signature  // 更新签名
  break
```

### 3.4 API参数构建

```typescript
// services/api/claude.ts:1594-1630
const hasThinking =
  thinkingConfig.type !== 'disabled' &&
  !isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_THINKING)

let thinking: BetaMessageStreamParams['thinking'] | undefined = undefined

if (hasThinking && modelSupportsThinking(options.model)) {
  if (
    !isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING) &&
    modelSupportsAdaptiveThinking(options.model)
  ) {
    // 自适应思考：无预算
    thinking = {
      type: 'adaptive',
    } satisfies BetaMessageStreamParams['thinking']
  } else {
    // 固定预算思考
    let thinkingBudget = getMaxThinkingTokensForModel(options.model)
    if (
      thinkingConfig.type === 'enabled' &&
      thinkingConfig.budgetTokens !== undefined
    ) {
      thinkingBudget = thinkingConfig.budgetTokens
    }
    thinkingBudget = Math.min(maxOutputTokens - 1, thinkingBudget)
    thinking = {
      budget_tokens: thinkingBudget,
      type: 'enabled',
    } satisfies BetaMessageStreamParams['thinking']
  }
}
```

**思考配置逻辑流程**：
1. 检查是否禁用
2. 检查模型是否支持
3. 选择adaptive或budget模式
4. budget必须小于maxOutputTokens-1

### 3.5 Temperature特殊处理

```typescript
// services/api/claude.ts:1691-1695
// Only send temperature when thinking is disabled — the API requires
// temperature: 1 when thinking is enabled, which is already the default.
const temperature = !hasThinking
  ? (options.temperatureOverride ?? 1)
  : undefined
```

**API约束**：thinking启用时temperature必须为1，所以不发送temperature参数。

---

## 4. 扩展思考

### 4.1 Redacted Thinking（加密思考）

```typescript
// utils/betas.ts:264-277
// Skip the API-side Haiku thinking summarizer — the summary is only used
// for ctrl+o display, which interactive users rarely open. The API returns
// redacted_thinking blocks instead; AssistantRedactedThinkingMessage already
// renders those as a stub.
if (
  includeFirstPartyOnlyBetas &&
  modelSupportsISP(model) &&
  !getIsNonInteractiveSession() &&
  getInitialSettings().showThinkingSummaries !== true
) {
  betaHeaders.push(REDACT_THINKING_BETA_HEADER)
}
```

**Redacted Thinking特性**：
- 使用beta header: `redact-thinking-2026-02-12`
- 返回加密的thinking块而非完整思考内容
- 减少token消耗，保护思考隐私
- 用户可通过`showThinkingSummaries: true`启用完整显示

```typescript
// constants/betas.ts:20
export const REDACT_THINKING_BETA_HEADER = 'redact-thinking-2026-02-12'
```

### 4.2 Interleaved Thinking（交错思考）

```typescript
// utils/betas.ts:257-262
if (
  !isEnvTruthy(process.env.DISABLE_INTERLEAVED_THINKING) &&
  modelSupportsISP(model)
) {
  betaHeaders.push(INTERLEAVED_THINKING_BETA_HEADER)
}
```

```typescript
// constants/betas.ts:4-5
export const INTERLEAVED_THINKING_BETA_HEADER =
  'interleaved-thinking-2025-05-14'
```

**Interleaved Thinking**：允许thinking块与其他内容块交错出现，而非必须连续。

### 4.3 Context Management（上下文管理）

```typescript
// services/api/claude.ts:1633-1637
const contextManagement = getAPIContextManagement({
  hasThinking,
  isRedactThinkingActive: betasParams.includes(REDACT_THINKING_BETA_HEADER),
  clearAllThinking: thinkingClearLatched,
})
```

```typescript
// utils/betas.ts:125-132
export function modelSupportsContextManagement(model: string): boolean {
  const canonical = getCanonicalName(model)
  const provider = getAPIProvider()
  if (provider === 'foundry') {
    return true
  }
  // Claude 4+支持上下文管理
  return !canonical.includes('claude-3-')
}
```

**Context Management特性**：
- beta header: `context-management-2025-06-27`
- 支持thinking块的保留和清理策略
- Claude 4+模型支持

### 4.4 第三方模型能力覆盖

```typescript
// utils/model/modelSupportOverrides.ts:4-10
export type ModelCapabilityOverride =
  | 'effort'
  | 'max_effort'
  | 'thinking'
  | 'adaptive_thinking'
  | 'interleaved_thinking'
```

```typescript
// utils/model/modelSupportOverrides.ts:30-50
export const get3PModelCapabilityOverride = memoize(
  (model: string, capability: ModelCapabilityOverride): boolean | undefined => {
    if (getAPIProvider() === 'firstParty') {
      return undefined  // 第一方API不需要覆盖
    }
    const m = model.toLowerCase()
    for (const tier of TIERS) {
      const pinned = process.env[tier.modelEnvVar]
      const capabilities = process.env[tier.capabilitiesEnvVar]
      if (!pinned || capabilities === undefined) continue
      if (m !== pinned.toLowerCase()) continue
      return capabilities
        .toLowerCase()
        .split(',')
        .map(s => s.trim())
        .includes(capability)
    }
    return undefined
  },
  (model, capability) => `${model.toLowerCase()}:${capability}`,
)
```

**覆盖机制**：
- 通过环境变量配置第三方模型能力
- 格式：`ANTHROPIC_DEFAULT_*_MODEL_SUPPORTED_CAPABILITIES`
- 用逗号分隔的能力列表

---

## 5. 用户配置

### 5.1 设置项

```typescript
// tools/ConfigTool/supportedSettings.ts:107-112
alwaysThinkingEnabled: {
  source: 'settings',
  type: 'boolean',
  description: 'Enable extended thinking (false to disable)',
  appStateKey: 'thinkingEnabled',
}
```

```typescript
// utils/settings/types.ts:696-701
alwaysThinkingEnabled: z
  .boolean()
  .optional()
  .describe(
    'When false, thinking is disabled. When absent or true, thinking is ' +
      'enabled automatically for supported models.',
  )
```

**设置逻辑**：
- `false`：禁用thinking
- `undefined`或`true`：自动启用（对支持的模型）

### 5.2 环境变量

```typescript
// utils/managedEnvConstants.ts:159
'MAX_THINKING_TOKENS',  // 思考token预算
```

相关环境变量：
- `MAX_THINKING_TOKENS`：设置thinking预算
- `CLAUDE_CODE_DISABLE_THINKING`：完全禁用thinking
- `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING`：禁用自适应thinking
- `DISABLE_INTERLEAVED_THINKING`：禁用交错thinking

### 5.3 命令行参数（main.tsx）

```typescript
// main.tsx:2462-2488
if (options.thinking === 'adaptive' || options.thinking === 'enabled') {
  thinkingEnabled = true;
  thinkingConfig = { type: 'adaptive' };
} else if (options.thinking === 'disabled') {
  thinkingEnabled = false;
  thinkingConfig = { type: 'disabled' };
} else {
  const maxThinkingTokens = process.env.MAX_THINKING_TOKENS
    ? parseInt(process.env.MAX_THINKING_TOKENS, 10)
    : options.maxThinkingTokens;
  if (maxThinkingTokens !== undefined) {
    if (maxThinkingTokens > 0) {
      thinkingEnabled = true;
      thinkingConfig = {
        type: 'enabled',
        budgetTokens: maxThinkingTokens
      };
    } else if (maxThinkingTokens === 0) {
      thinkingEnabled = false;
      thinkingConfig = { type: 'disabled' };
    }
  }
}
```

### 5.4 UI切换组件

```typescript
// components/ThinkingToggle.tsx:12-17
export type Props = {
  currentValue: boolean;
  onSelect: (enabled: boolean) => void;
  onCancel?: () => void;
  isMidConversation?: boolean;  // 对话中途切换需要确认
};
```

```typescript
// components/ThinkingToggle.tsx:97-104
function handleSelectChange(value: string): void {
  const selected = value === 'true';
  if (isMidConversation && selected !== currentValue) {
    setConfirmationPending(selected);  // 中途切换需要确认
  } else {
    onSelect(selected);
  }
}
```

**中途切换警告**：
> Changing thinking mode mid-conversation will increase latency and may reduce quality. For best results, set this at the start of a session.

### 5.5 Thinking预算计算

```typescript
// utils/context.ts:212-221
/**
 * Returns the max thinking budget tokens for a given model. The max
 * thinking tokens should be strictly less than the max output tokens.
 *
 * Deprecated since newer models use adaptive thinking rather than a
 * strict thinking token budget.
 */
export function getMaxThinkingTokensForModel(model: string): number {
  return getModelMaxOutputTokens(model).upperLimit - 1
}
```

**预算约束**：thinking_budget必须严格小于max_output_tokens。

---

## 6. 从零实现

### 6.1 最简Thinking系统

```typescript
// === types.ts ===
export type ThinkingConfig =
  | { type: 'adaptive' }
  | { type: 'enabled'; budgetTokens: number }
  | { type: 'disabled' }

// === thinking.ts ===
export function modelSupportsThinking(model: string): boolean {
  // 简化版本：Claude 4+支持
  return model.includes('claude-4') || model.includes('opus-4') || model.includes('sonnet-4')
}

export function shouldEnableThinkingByDefault(): boolean {
  // 默认启用
  return true
}

// === api.ts ===
function buildThinkingParams(config: ThinkingConfig, model: string) {
  if (config.type === 'disabled') return undefined
  if (!modelSupportsThinking(model)) return undefined

  if (config.type === 'adaptive') {
    return { type: 'adaptive' }
  }

  // 固定预算模式
  const budget = Math.min(config.budgetTokens, getMaxOutputTokens(model) - 1)
  return { type: 'enabled', budget_tokens: budget }
}
```

### 6.2 流式处理思考块

```typescript
// === streamHandler.ts ===
interface ThinkingBlock {
  type: 'thinking'
  thinking: string
  signature: string
}

function handleContentBlockStart(event: StreamEvent): ContentBlock {
  if (event.content_block.type === 'thinking') {
    return {
      type: 'thinking',
      thinking: '',
      signature: '',
    }
  }
  // ... 其他块类型
}

function handleContentBlockDelta(block: ContentBlock, delta: Delta): void {
  if (delta.type === 'thinking_delta' && block.type === 'thinking') {
    block.thinking += delta.thinking
  }
  if (delta.type === 'signature_delta' && block.type === 'thinking') {
    block.signature = delta.signature
  }
}
```

### 6.3 UI渲染思考内容

```typescript
// === ThinkingMessage.tsx ===
interface Props {
  thinking: string
  isExpanded: boolean
}

export function ThinkingMessage({ thinking, isExpanded }: Props) {
  if (!thinking) return null

  if (!isExpanded) {
    return (
      <Text dimColor italic>
        ∴ Thinking <CtrlOToExpand />
      </Text>
    )
  }

  return (
    <Box flexDirection="column">
      <Text dimColor italic>∴ Thinking…</Text>
      <Box paddingLeft={2}>
        <Markdown dimColor>{thinking}</Markdown>
      </Box>
    </Box>
  )
}
```

### 6.4 Rainbow高亮实现

```typescript
// === rainbowColors.ts ===
const RAINBOW_COLORS = [
  'rainbow_red', 'rainbow_orange', 'rainbow_yellow',
  'rainbow_green', 'rainbow_blue', 'rainbow_indigo', 'rainbow_violet',
]

export function getRainbowColor(index: number): string {
  return RAINBOW_COLORS[index % RAINBOW_COLORS.length]
}

// === render.tsx ===
function renderRainbowText(text: string) {
  return text.split('').map((char, i) => (
    <Text key={i} color={getRainbowColor(i)}>{char}</Text>
  ))
}
```

### 6.5 Ultrathink关键词检测

```typescript
// === ultrathinkDetection.ts ===
export function hasUltrathinkKeyword(text: string): boolean {
  return /\bultrathink\b/i.test(text)
}

export function findUltrathinkPositions(text: string): Position[] {
  const positions: Position[] = []
  const regex = /\bultrathink\b/gi  // 每次新正则避免状态泄漏

  for (const match of text.matchAll(regex)) {
    positions.push({
      word: match[0],
      start: match.index!,
      end: match.index! + match[0].length,
    })
  }

  return positions
}
```

### 6.6 完整集成示例

```typescript
// === thinkingIntegration.ts ===
import { ThinkingConfig, modelSupportsThinking, modelSupportsAdaptiveThinking } from './thinking'
import { getSettings } from './settings'

export function configureThinking(model: string, options: CLIOptions): ThinkingConfig {
  // 1. 命令行参数优先
  if (options.thinking === 'disabled') {
    return { type: 'disabled' }
  }
  if (options.thinking === 'adaptive' || options.thinking === 'enabled') {
    return { type: 'adaptive' }
  }

  // 2. 环境变量
  if (process.env.MAX_THINKING_TOKENS) {
    const tokens = parseInt(process.env.MAX_THINKING_TOKENS, 10)
    if (tokens <= 0) return { type: 'disabled' }
    return { type: 'enabled', budgetTokens: tokens }
  }

  // 3. 用户设置
  const settings = getSettings()
  if (settings.alwaysThinkingEnabled === false) {
    return { type: 'disabled' }
  }

  // 4. 模型能力检测
  if (!modelSupportsThinking(model)) {
    return { type: 'disabled' }
  }

  // 5. 自适应vs固定预算
  if (modelSupportsAdaptiveThinking(model)) {
    return { type: 'adaptive' }
  }

  // 6. 默认固定预算
  const defaultBudget = getMaxThinkingTokens(model)
  return { type: 'enabled', budgetTokens: defaultBudget }
}

// API调用
async function callAPI(messages: Message[], config: ThinkingConfig, model: string) {
  const thinkingParam = buildThinkingParams(config, model)

  return client.beta.messages.create({
    model,
    messages,
    max_tokens: getMaxOutputTokens(model),
    ...(thinkingParam && { thinking: thinkingParam }),
    // thinking启用时temperature默认为1，不发送
    ...(thinkingParam?.type === 'disabled' && { temperature: 1 }),
  })
}
```

---

## 7. 关键设计要点总结

### 7.1 多层配置优先级

1. 命令行参数（最高）
2. 环境变量
3. 用户设置
4. 模型能力检测
5. 默认值

### 7.2 模型能力矩阵

| 模型系列 | Thinking | Adaptive Thinking | Context Management |
|---------|----------|-------------------|-------------------|
| Claude 3 | No | No | No |
| Claude 4.0-4.5 | Yes | No | Yes |
| Claude 4.6+ | Yes | Yes | Yes |

### 7.3 API约束

- `thinking`启用时`temperature`必须为1
- `budget_tokens`必须小于`max_tokens - 1`
- thinking块不能是消息最后一个块

### 7.4 Beta Headers

| 特性 | Header |
|-----|--------|
| Interleaved Thinking | `interleaved-thinking-2025-05-14` |
| Redacted Thinking | `redact-thinking-2026-02-12` |
| Context Management | `context-management-2025-06-27` |

### 7.5 关键文件路径

- 核心逻辑：`utils/thinking.ts`
- API处理：`services/api/claude.ts`
- Beta配置：`utils/betas.ts`
- UI组件：`components/ThinkingToggle.tsx`, `components/messages/AssistantThinkingMessage.tsx`
- 设置定义：`utils/settings/types.ts`, `tools/ConfigTool/supportedSettings.ts`
- 模型支持：`utils/model/modelSupportOverrides.ts`

---

*Thinking系统分析完成*