# 81. API成本计算详解

**分析日期**: 2026-04-02

## 1. 成本计算概述

### 1.1 核心架构

Claude Code的成本计算系统采用三层架构：

```
┌─────────────────────────────────────────────────────────────┐
│                    用户层 (/cost命令)                         │
│  commands/cost/cost.ts → formatTotalCost()                   │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                    追踪层 (cost-tracker.ts)                   │
│  addToTotalSessionCost() → addToTotalModelUsage()            │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                    计算层 (utils/modelCost.ts)                │
│  calculateUSDCost() → tokensToUSDCost() → MODEL_COSTS         │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 数据流向

每次API调用后的成本计算流程：

```typescript
// 1. API返回Usage数据
const usage: BetaUsage = {
  input_tokens: 5000,
  output_tokens: 1500,
  cache_read_input_tokens: 2000,
  cache_creation_input_tokens: 500,
  server_tool_use: { web_search_requests: 3 }
}

// 2. 计算成本 (modelCost.ts)
const cost = calculateUSDCost('claude-sonnet-4-6', usage)

// 3. 更新状态 (bootstrap/state.ts)
STATE.totalCostUSD += cost
STATE.modelUsage[model] = modelUsage

// 4. 更新遥测 (cost-tracker.ts)
getCostCounter()?.add(cost, { model, speed: 'fast' })
```

### 1.3 核心文件

| 文件路径 | 职责 |
|---------|------|
| `utils/modelCost.ts` | **定价配置与成本计算** |
| `cost-tracker.ts` | **成本追踪与展示** |
| `bootstrap/state.ts` | **状态存储** |
| `commands/cost/cost.ts` | **用户命令实现** |
| `costHook.ts` | **退出时保存成本** |

---

## 2. Token定价表

### 2.1 定价常量定义

**文件位置**: `utils/modelCost.ts` 第36-89行

```typescript
// 标准定价层: $3输入/$15输出 每百万Token
export const COST_TIER_3_15 = {
  inputTokens: 3,                      // $3/M tokens
  outputTokens: 15,                    // $15/M tokens
  promptCacheWriteTokens: 3.75,        // $3.75/M tokens (1.25x input)
  promptCacheReadTokens: 0.3,          // $0.30/M tokens (0.1x input)
  webSearchRequests: 0.01,             // $0.01/request
} as const satisfies ModelCosts

// Opus 4/4.1定价层: $15输入/$75输出
export const COST_TIER_15_75 = {
  inputTokens: 15,
  outputTokens: 75,
  promptCacheWriteTokens: 18.75,       // 1.25x input
  promptCacheReadTokens: 1.5,          // 0.1x input
  webSearchRequests: 0.01,
} as const satisfies ModelCosts

// Opus 4.5定价层: $5输入/$25输出
export const COST_TIER_5_25 = {
  inputTokens: 5,
  outputTokens: 25,
  promptCacheWriteTokens: 6.25,
  promptCacheReadTokens: 0.5,
  webSearchRequests: 0.01,
} as const satisfies ModelCosts

// Opus 4.6快速模式: $30输入/$150输出
export const COST_TIER_30_150 = {
  inputTokens: 30,
  outputTokens: 150,
  promptCacheWriteTokens: 37.5,
  promptCacheReadTokens: 3,
  webSearchRequests: 0.01,
} as const satisfies ModelCosts

// Haiku 3.5: $0.80输入/$4输出
export const COST_HAIKU_35 = {
  inputTokens: 0.8,
  outputTokens: 4,
  promptCacheWriteTokens: 1,
  promptCacheReadTokens: 0.08,
  webSearchRequests: 0.01,
} as const satisfies ModelCosts

// Haiku 4.5: $1输入/$5输出
export const COST_HAIKU_45 = {
  inputTokens: 1,
  outputTokens: 5,
  promptCacheWriteTokens: 1.25,
  promptCacheReadTokens: 0.1,
  webSearchRequests: 0.01,
} as const satisfies ModelCosts
```

### 2.2 各模型定价映射表

**文件位置**: `utils/modelCost.ts` 第104-126行

```typescript
export const MODEL_COSTS: Record<ModelShortName, ModelCosts> = {
  // Haiku系列 - 轻量级模型
  'claude-3-5-haiku': COST_HAIKU_35,        // $0.80/$4
  'claude-haiku-4-5': COST_HAIKU_45,        // $1/$5

  // Sonnet系列 - 标准模型
  'claude-3-5-sonnet': COST_TIER_3_15,      // $3/$15
  'claude-3-7-sonnet': COST_TIER_3_15,      // $3/$15
  'claude-sonnet-4': COST_TIER_3_15,        // $3/$15
  'claude-sonnet-4-5': COST_TIER_3_15,      // $3/$15
  'claude-sonnet-4-6': COST_TIER_3_15,      // $3/$15

  // Opus系列 - 高级模型
  'claude-opus-4': COST_TIER_15_75,         // $15/$75
  'claude-opus-4-1': COST_TIER_15_75,       // $15/$75
  'claude-opus-4-5': COST_TIER_5_25,        // $5/$25
  'claude-opus-4-6': COST_TIER_5_25,        // $5/$25 (标准模式)
}
```

### 2.3 完整定价表（美元/百万Token）

| 模型 | 输入价格 | 输出价格 | 缓存写入 | 缓存读取 | Web搜索 |
|------|---------|---------|---------|---------|---------|
| **Haiku 3.5** | $0.80 | $4 | $1 | $0.08 | $0.01 |
| **Haiku 4.5** | $1 | $5 | $1.25 | $0.10 | $0.01 |
| **Sonnet 3.5/3.7/4.x** | $3 | $15 | $3.75 | $0.30 | $0.01 |
| **Opus 4/4.1** | $15 | $75 | $18.75 | $1.50 | $0.01 |
| **Opus 4.5** | $5 | $25 | $6.25 | $0.50 | $0.01 |
| **Opus 4.6 (标准)** | $5 | $25 | $6.25 | $0.50 | $0.01 |
| **Opus 4.6 (快速)** | $30 | $150 | $37.50 | $3.00 | $0.01 |

---

## 3. 输入/输出成本计算

### 3.1 成本计算核心函数

**文件位置**: `utils/modelCost.ts` 第131-142行

```typescript
/**
 * 将Token使用量转换为美元成本
 * 核心公式实现
 */
function tokensToUSDCost(modelCosts: ModelCosts, usage: Usage): number {
  return (
    // 输入Token成本 (每百万Token的价格)
    (usage.input_tokens / 1_000_000) * modelCosts.inputTokens +

    // 输出Token成本
    (usage.output_tokens / 1_000_000) * modelCosts.outputTokens +

    // 缓存读取Token成本 (90%折扣)
    ((usage.cache_read_input_tokens ?? 0) / 1_000_000) *
      modelCosts.promptCacheReadTokens +

    // 缓存创建Token成本 (25%溢价)
    ((usage.cache_creation_input_tokens ?? 0) / 1_000_000) *
      modelCosts.promptCacheWriteTokens +

    // Web搜索请求成本
    (usage.server_tool_use?.web_search_requests ?? 0) *
      modelCosts.webSearchRequests
  )
}
```

### 3.2 成本计算入口函数

**文件位置**: `utils/modelCost.ts` 第177-180行

```typescript
/**
 * 计算API调用的美元成本
 * @param resolvedModel 模型名称（已解析）
 * @param usage API返回的Token使用数据
 * @returns 美元成本
 */
export function calculateUSDCost(resolvedModel: string, usage: Usage): number {
  const modelCosts = getModelCosts(resolvedModel, usage)
  return tokensToUSDCost(modelCosts, usage)
}
```

### 3.3 获取模型定价配置

**文件位置**: `utils/modelCost.ts` 第144-164行

```typescript
export function getModelCosts(model: string, usage: Usage): ModelCosts {
  const shortName = getCanonicalName(model)

  // Opus 4.6特殊处理：根据speed参数动态选择定价层
  if (shortName === 'claude-opus-4-6') {
    const isFastMode = usage.speed === 'fast'
    return getOpus46CostTier(isFastMode)  // 快速模式用COST_TIER_30_150
  }

  // 查找模型定价配置
  const costs = MODEL_COSTS[shortName]
  if (!costs) {
    // 未知模型：记录事件并使用默认定价
    trackUnknownModelCost(model, shortName)
    return MODEL_COSTS[getCanonicalName(getDefaultMainLoopModelSetting())]
      ?? DEFAULT_UNKNOWN_MODEL_COST  // 默认使用COST_TIER_5_25
  }
  return costs
}
```

### 3.4 实际计算示例

```typescript
// 示例：Sonnet 4.6的API调用
const usage = {
  input_tokens: 10000,              // 10K输入
  output_tokens: 3000,              // 3K输出
  cache_read_input_tokens: 5000,    // 5K缓存读取
  cache_creation_input_tokens: 0,   // 无新缓存
  server_tool_use: { web_search_requests: 2 }
}

// 计算：
// 输入成本: (10000/1M) × $3 = $0.03
// 输出成本: (3000/1M) × $15 = $0.045
// 缓存读取: (5000/1M) × $0.30 = $0.015
// 缓存创建: (0/1M) × $3.75 = $0
// Web搜索: 2 × $0.01 = $0.02
// 总成本: $0.03 + $0.045 + $0.015 + $0 + $0.02 = $0.11
```

---

## 4. 缓存折扣机制

### 4.1 缓存定价原理

Prompt Caching允许缓存重复使用的提示词部分，实现显著成本节约：

| Token类型 | 价格系数 | 说明 |
|----------|---------|------|
| **普通输入** | 1.0x | 无缓存，全额计费 |
| **缓存写入** | 1.25x | 首次写入，略微溢价 |
| **缓存读取** | 0.1x | 读取缓存，90%折扣 |

### 4.2 缓存成本对比示例

```typescript
// 无缓存的请求
const noCacheUsage = {
  input_tokens: 100000,              // 全额输入
  cache_read_input_tokens: 0,
  cache_creation_input_tokens: 0
}
// Sonnet 4.6成本: (100000/1M) × $3 = $0.30

// 有缓存的请求（50%命中）
const cachedUsage = {
  input_tokens: 50000,               // 新输入部分
  cache_read_input_tokens: 50000,    // 缓存命中部分
  cache_creation_input_tokens: 0
}
// 成本计算:
// 新输入: (50000/1M) × $3 = $0.15
// 缓存读取: (50000/1M) × $0.30 = $0.015
// 总成本: $0.165 (节省45%)
```

### 4.3 缓存写入成本

```typescript
// 首次请求（需要创建缓存）
const firstRequest = {
  input_tokens: 50000,
  cache_creation_input_tokens: 50000  // 创建50K缓存
}
// 成本:
// 输入: (50000/1M) × $3 = $0.15
// 缓存写入: (50000/1M) × $3.75 = $0.1875
// 总计: $0.3375 (首次略贵，后续请求省钱)

// 第二次请求（缓存命中）
const secondRequest = {
  input_tokens: 10000,               // 仅新增部分
  cache_read_input_tokens: 50000     // 缓存命中
}
// 成本:
// 新输入: (10000/1M) × $3 = $0.03
// 缓存读取: (50000/1M) × $0.30 = $0.015
// 总计: $0.045 (大幅节省)

// 两次合计: $0.3375 + $0.045 = $0.3825
// 对比无缓存两次: $0.30 + $0.30 = $0.60
// 总节省: 36%
```

### 4.4 缓存相关状态追踪

**文件位置**: `bootstrap/state.ts` 第157-158行

```typescript
// 状态中的缓存Token计数
STATE.totalCacheReadInputTokens = 0      // 累计缓存读取
STATE.totalCacheCreationInputTokens = 0  // 累计缓存创建
```

---

## 5. 多模型统计

### 5.1 ModelUsage数据结构

**文件位置**: `entrypoints/sdk/coreSchemas.ts` 第17-27行

```typescript
export const ModelUsageSchema = z.object({
  inputTokens: z.number(),               // 输入Token累计
  outputTokens: z.number(),              // 输出Token累计
  cacheReadInputTokens: z.number(),      // 缓存读取累计
  cacheCreationInputTokens: z.number(),  // 缓存创建累计
  webSearchRequests: z.number(),         // Web搜索次数
  costUSD: z.number(),                   // 该模型总成本
  contextWindow: z.number(),             // 上下文窗口大小
  maxOutputTokens: z.number(),           // 最大输出Token
})
```

### 5.2 状态存储结构

**文件位置**: `bootstrap/state.ts` 第67行

```typescript
// 全局状态中的模型使用映射
modelUsage: { [modelName: string]: ModelUsage }

// 示例数据结构：
STATE.modelUsage = {
  'claude-sonnet-4-6': {
    inputTokens: 50000,
    outputTokens: 15000,
    cacheReadInputTokens: 30000,
    cacheCreationInputTokens: 10000,
    webSearchRequests: 5,
    costUSD: 0.35,
    contextWindow: 200000,
    maxOutputTokens: 8192
  },
  'claude-opus-4-6': {
    inputTokens: 20000,
    outputTokens: 8000,
    ...
    costUSD: 0.25
  }
}
```

### 5.3 更新模型使用统计

**文件位置**: `cost-tracker.ts` 第250-276行

```typescript
/**
 * 更新单个模型的使用统计
 * @param cost 本次成本
 * @param usage 本次Token使用
 * @param model 模型名称
 * @returns 更新后的ModelUsage
 */
function addToTotalModelUsage(
  cost: number,
  usage: Usage,
  model: string,
): ModelUsage {
  // 获取或创建该模型的统计记录
  const modelUsage = getUsageForModel(model) ?? {
    inputTokens: 0,
    outputTokens: 0,
    cacheReadInputTokens: 0,
    cacheCreationInputTokens: 0,
    webSearchRequests: 0,
    costUSD: 0,
    contextWindow: 0,
    maxOutputTokens: 0,
  }

  // 累加各项统计
  modelUsage.inputTokens += usage.input_tokens
  modelUsage.outputTokens += usage.output_tokens
  modelUsage.cacheReadInputTokens += usage.cache_read_input_tokens ?? 0
  modelUsage.cacheCreationInputTokens += usage.cache_creation_input_tokens ?? 0
  modelUsage.webSearchRequests += usage.server_tool_use?.web_search_requests ?? 0
  modelUsage.costUSD += cost

  // 更新模型元数据
  modelUsage.contextWindow = getContextWindowForModel(model, getSdkBetas())
  modelUsage.maxOutputTokens = getModelMaxOutputTokens(model).default

  return modelUsage
}
```

### 5.4 添加会话总成本

**文件位置**: `cost-tracker.ts` 第278-323行

```typescript
/**
 * 添加本次API调用的成本到会话统计
 * @param cost 本次成本
 * @param usage Token使用数据
 * @param model 模型名称
 * @returns 总成本（包括advisor成本）
 */
export function addToTotalSessionCost(
  cost: number,
  usage: Usage,
  model: string,
): number {
  // 1. 更新模型使用统计
  const modelUsage = addToTotalModelUsage(cost, usage, model)
  addToTotalCostState(cost, modelUsage, model)

  // 2. 构建遥测属性
  const attrs = isFastModeEnabled() && usage.speed === 'fast'
    ? { model, speed: 'fast' }
    : { model }

  // 3. 更新遥测计数器
  getCostCounter()?.add(cost, attrs)
  getTokenCounter()?.add(usage.input_tokens, { ...attrs, type: 'input' })
  getTokenCounter()?.add(usage.output_tokens, { ...attrs, type: 'output' })
  getTokenCounter()?.add(usage.cache_read_input_tokens ?? 0, { ...attrs, type: 'cacheRead' })
  getTokenCounter()?.add(usage.cache_creation_input_tokens ?? 0, { ...attrs, type: 'cacheCreation' })

  // 4. 处理Advisor工具成本（递归）
  let totalCost = cost
  for (const advisorUsage of getAdvisorUsage(usage)) {
    const advisorCost = calculateUSDCost(advisorUsage.model, advisorUsage)
    logEvent('tengu_advisor_tool_token_usage', {
      advisor_model: advisorUsage.model,
      input_tokens: advisorUsage.input_tokens,
      output_tokens: advisorUsage.output_tokens,
      cache_read_input_tokens: advisorUsage.cache_read_input_tokens ?? 0,
      cache_creation_input_tokens: advisorUsage.cache_creation_input_tokens ?? 0,
      cost_usd_micros: Math.round(advisorCost * 1_000_000),
    })
    // 递归添加advisor成本
    totalCost += addToTotalSessionCost(advisorCost, advisorUsage, advisorUsage.model)
  }

  return totalCost
}
```

### 5.5 模型名称规范化

**文件位置**: `utils/model/model.ts` 第217-270行

```typescript
/**
 * 将完整模型名映射到短名称
 * 统一不同提供商的模型命名
 */
export function firstPartyNameToCanonical(name: ModelName): ModelShortName {
  name = name.toLowerCase()

  // Claude 4+模型检查（顺序很重要：先检查更具体的版本）
  if (name.includes('claude-opus-4-6')) return 'claude-opus-4-6'
  if (name.includes('claude-opus-4-5')) return 'claude-opus-4-5'
  if (name.includes('claude-opus-4-1')) return 'claude-opus-4-1'
  if (name.includes('claude-opus-4')) return 'claude-opus-4'

  if (name.includes('claude-sonnet-4-6')) return 'claude-sonnet-4-6'
  if (name.includes('claude-sonnet-4-5')) return 'claude-sonnet-4-5'
  if (name.includes('claude-sonnet-4')) return 'claude-sonnet-4'

  if (name.includes('claude-haiku-4-5')) return 'claude-haiku-4-5'

  // Claude 3.x模型
  if (name.includes('claude-3-7-sonnet')) return 'claude-3-7-sonnet'
  if (name.includes('claude-3-5-sonnet')) return 'claude-3-5-sonnet'
  if (name.includes('claude-3-5-haiku')) return 'claude-3-5-haiku'

  // 回退：正则匹配
  const match = name.match(/(claude-(\d+-\d+-)?\w+)/)
  return match?.[1] ?? name
}
```

---

## 6. 用户使用指南

### 6.1 查看当前成本

```bash
# 在Claude Code会话中使用命令
/cost

# 输出示例：
Total cost:            $0.3542
Total duration (API):  12.5s
Total duration (wall): 1m 30s
Total code changes:    25 lines added, 10 lines removed
Usage by model:
          claude-sonnet-4-6:  50,000 input, 15,000 output, 30,000 cache read, 10,000 cache write ($0.30)
          claude-opus-4-6:  20,000 input, 8,000 output, 5,000 cache read, 0 cache write ($0.15)
```

### 6.2 成本显示条件

**文件位置**: `utils/billing.ts` 第10-44行

```typescript
/**
 * 判断用户是否有权限查看成本
 */
export function hasConsoleBillingAccess(): boolean {
  // 1. 禁用成本警告时不显示
  if (isEnvTruthy(process.env.DISABLE_COST_WARNINGS)) {
    return false
  }

  // 2. Claude AI订阅者不显示（无限额度）
  if (isClaudeAISubscriber()) return false

  // 3. 无认证用户不显示
  const authSource = getAuthTokenSource()
  const hasApiKey = getAnthropicApiKey() !== null
  if (!authSource.hasToken && !hasApiKey) {
    return false
  }

  // 4. 需要有组织/工作区角色
  const orgRole = config.oauthAccount?.organizationRole
  const workspaceRole = config.oauthAccount?.workspaceRole
  if (!orgRole || !workspaceRole) {
    return false
  }

  // 5. 仅管理员和计费角色可查看
  return (
    ['admin', 'billing'].includes(orgRole) ||
    ['workspace_admin', 'workspace_billing'].includes(workspaceRole)
  )
}
```

### 6.3 成本格式化

**文件位置**: `cost-tracker.ts` 第177-179行

```typescript
/**
 * 格式化成本显示
 * @param cost 成本值
 * @param maxDecimalPlaces 最大小数位
 */
function formatCost(cost: number, maxDecimalPlaces: number = 4): string {
  // 超过$0.5显示2位小数，否则显示4位
  return `$${cost > 0.5
    ? round(cost, 100).toFixed(2)
    : cost.toFixed(maxDecimalPlaces)}`
}

// 示例输出：
// $0.3542 → "$0.3542" (小于$0.5)
// $1.2345 → "$1.23"   (大于$0.5)
```

### 6.4 模型使用格式化

**文件位置**: `cost-tracker.ts` 第181-226行

```typescript
/**
 * 格式化模型使用统计
 * 按短名称聚合多个模型版本
 */
function formatModelUsage(): string {
  const modelUsageMap = getModelUsage()
  if (Object.keys(modelUsageMap).length === 0) {
    return 'Usage:                 0 input, 0 output, 0 cache read, 0 cache write'
  }

  // 按短名称聚合
  const usageByShortName: { [shortName: string]: ModelUsage } = {}
  for (const [model, usage] of Object.entries(modelUsageMap)) {
    const shortName = getCanonicalName(model)
    if (!usageByShortName[shortName]) {
      usageByShortName[shortName] = createEmptyUsage()
    }
    // 累加到短名称
    const accumulated = usageByShortName[shortName]
    accumulated.inputTokens += usage.inputTokens
    accumulated.outputTokens += usage.outputTokens
    accumulated.cacheReadInputTokens += usage.cacheReadInputTokens
    accumulated.cacheCreationInputTokens += usage.cacheCreationInputTokens
    accumulated.webSearchRequests += usage.webSearchRequests
    accumulated.costUSD += usage.costUSD
  }

  // 格式化输出
  let result = 'Usage by model:'
  for (const [shortName, usage] of Object.entries(usageByShortName)) {
    const usageString =
      `  ${formatNumber(usage.inputTokens)} input, ` +
      `${formatNumber(usage.outputTokens)} output, ` +
      `${formatNumber(usage.cacheReadInputTokens)} cache read, ` +
      `${formatNumber(usage.cacheCreationInputTokens)} cache write` +
      (usage.webSearchRequests > 0
        ? `, ${formatNumber(usage.webSearchRequests)} web search`
        : '') +
      ` (${formatCost(usage.costUSD)})`
    result += `\n` + `${shortName}:`.padStart(21) + usageString
  }
  return result
}
```

### 6.5 总成本格式化

**文件位置**: `cost-tracker.ts` 第228-244行

```typescript
/**
 * 格式化完整的成本报告
 */
export function formatTotalCost(): string {
  const costDisplay =
    formatCost(getTotalCostUSD()) +
    (hasUnknownModelCost()
      ? ' (costs may be inaccurate due to usage of unknown models)'
      : '')

  const modelUsageDisplay = formatModelUsage()

  return chalk.dim(
    `Total cost:            ${costDisplay}\n` +
    `Total duration (API):  ${formatDuration(getTotalAPIDuration())}
Total duration (wall): ${formatDuration(getTotalDuration())}
Total code changes:    ${getTotalLinesAdded()} ${getTotalLinesAdded() === 1 ? 'line' : 'lines'} added, ${getTotalLinesRemoved()} ${getTotalLinesRemoved() === 1 ? 'line' : 'lines'} removed
${modelUsageDisplay}`,
  )
}
```

---

## 7. 从零实现

### 7.1 最简成本计算器

```typescript
// === 1. 定义定价结构 ===
interface ModelCosts {
  inputTokens: number      // $/M input
  outputTokens: number     // $/M output
  promptCacheWriteTokens: number  // $/M cache write
  promptCacheReadTokens: number   // $/M cache read
  webSearchRequests: number       // $/request
}

// === 2. 定义定价表 ===
const MODEL_COSTS: Record<string, ModelCosts> = {
  'claude-sonnet-4-6': {
    inputTokens: 3,
    outputTokens: 15,
    promptCacheWriteTokens: 3.75,
    promptCacheReadTokens: 0.3,
    webSearchRequests: 0.01,
  },
  'claude-opus-4-6': {
    inputTokens: 5,
    outputTokens: 25,
    promptCacheWriteTokens: 6.25,
    promptCacheReadTokens: 0.5,
    webSearchRequests: 0.01,
  },
}

// === 3. 定义Usage结构 ===
interface Usage {
  input_tokens: number
  output_tokens: number
  cache_read_input_tokens?: number
  cache_creation_input_tokens?: number
  server_tool_use?: { web_search_requests: number }
}

// === 4. 核心计算函数 ===
function calculateCost(model: string, usage: Usage): number {
  const costs = MODEL_COSTS[model] ?? MODEL_COSTS['claude-sonnet-4-6']

  return (
    (usage.input_tokens / 1_000_000) * costs.inputTokens +
    (usage.output_tokens / 1_000_000) * costs.outputTokens +
    ((usage.cache_read_input_tokens ?? 0) / 1_000_000) * costs.promptCacheReadTokens +
    ((usage.cache_creation_input_tokens ?? 0) / 1_000_000) * costs.promptCacheWriteTokens +
    (usage.server_tool_use?.web_search_requests ?? 0) * costs.webSearchRequests
  )
}

// === 5. 测试 ===
const usage: Usage = {
  input_tokens: 10000,
  output_tokens: 3000,
  cache_read_input_tokens: 5000,
  cache_creation_input_tokens: 0,
}

const cost = calculateCost('claude-sonnet-4-6', usage)
console.log(`Cost: $${cost.toFixed(4)}`)  // Cost: $0.0645
```

### 7.2 完整成本追踪系统

```typescript
// === 1. 状态管理 ===
interface CostState {
  totalCostUSD: number
  modelUsage: Record<string, ModelUsage>
}

interface ModelUsage {
  inputTokens: number
  outputTokens: number
  cacheReadInputTokens: number
  cacheCreationInputTokens: number
  webSearchRequests: number
  costUSD: number
}

const state: CostState = {
  totalCostUSD: 0,
  modelUsage: {},
}

// === 2. 状态操作函数 ===
function getTotalCost(): number {
  return state.totalCostUSD
}

function getModelUsage(): Record<string, ModelUsage> {
  return state.modelUsage
}

function getUsageForModel(model: string): ModelUsage | undefined {
  return state.modelUsage[model]
}

function resetCostState(): void {
  state.totalCostUSD = 0
  state.modelUsage = {}
}

// === 3. 添加成本 ===
function addCost(cost: number, usage: Usage, model: string): void {
  // 更新模型统计
  const modelUsage = getUsageForModel(model) ?? createEmptyUsage()
  modelUsage.inputTokens += usage.input_tokens
  modelUsage.outputTokens += usage.output_tokens
  modelUsage.cacheReadInputTokens += usage.cache_read_input_tokens ?? 0
  modelUsage.cacheCreationInputTokens += usage.cache_creation_input_tokens ?? 0
  modelUsage.webSearchRequests += usage.server_tool_use?.web_search_requests ?? 0
  modelUsage.costUSD += cost

  // 更新状态
  state.modelUsage[model] = modelUsage
  state.totalCostUSD += cost
}

function createEmptyUsage(): ModelUsage {
  return {
    inputTokens: 0,
    outputTokens: 0,
    cacheReadInputTokens: 0,
    cacheCreationInputTokens: 0,
    webSearchRequests: 0,
    costUSD: 0,
  }
}

// === 4. 模拟API调用处理 ===
function handleAPIResponse(model: string, usage: Usage): void {
  const cost = calculateCost(model, usage)
  addCost(cost, usage, model)
  console.log(`API call cost: $${cost.toFixed(4)}`)
  console.log(`Session total: $${getTotalCost().toFixed(4)}`)
}

// === 5. 格式化输出 ===
function formatCostReport(): string {
  const lines: string[] = []

  lines.push(`Total cost: $${getTotalCost().toFixed(4)}`)
  lines.push(`Usage by model:`)

  for (const [model, usage] of Object.entries(getModelUsage())) {
    lines.push(
      `  ${model}: ` +
      `${usage.inputTokens} input, ` +
      `${usage.outputTokens} output, ` +
      `${usage.cacheReadInputTokens} cache read, ` +
      `${usage.cacheCreationInputTokens} cache write ` +
      `($${usage.costUSD.toFixed(4)})`
    )
  }

  return lines.join('\n')
}

// === 6. 完整测试流程 ===
// 第一次API调用
handleAPIResponse('claude-sonnet-4-6', {
  input_tokens: 50000,
  output_tokens: 15000,
  cache_read_input_tokens: 0,
  cache_creation_input_tokens: 50000,
})
// API call cost: $0.3375
// Session total: $0.3375

// 第二次API调用（缓存命中）
handleAPIResponse('claude-sonnet-4-6', {
  input_tokens: 10000,
  output_tokens: 5000,
  cache_read_input_tokens: 50000,
  cache_creation_input_tokens: 0,
})
// API call cost: $0.075
// Session total: $0.4125

// 第三次API调用（不同模型）
handleAPIResponse('claude-opus-4-6', {
  input_tokens: 20000,
  output_tokens: 8000,
  cache_read_input_tokens: 0,
  cache_creation_input_tokens: 0,
})
// API call cost: $0.300
// Session total: $0.7125

console.log(formatCostReport())
// Total cost: $0.7125
// Usage by model:
//   claude-sonnet-4-6: 60000 input, 20000 output, 50000 cache read, 50000 cache write ($0.4125)
//   claude-opus-4-6: 20000 input, 8000 output, 0 cache read, 0 cache write ($0.300)
```

### 7.3 带会话持久化的实现

```typescript
// === 1. 项目配置存储 ===
interface ProjectConfig {
  lastCost?: number
  lastModelUsage?: Record<string, {
    inputTokens: number
    outputTokens: number
    cacheReadInputTokens: number
    cacheCreationInputTokens: number
    webSearchRequests: number
    costUSD: number
  }>
  lastSessionId?: string
}

// === 2. 保存会话成本 ===
function saveSessionCosts(sessionId: string): void {
  const config: ProjectConfig = {
    lastCost: getTotalCost(),
    lastModelUsage: Object.fromEntries(
      Object.entries(getModelUsage()).map(([model, usage]) => [
        model,
        {
          inputTokens: usage.inputTokens,
          outputTokens: usage.outputTokens,
          cacheReadInputTokens: usage.cacheReadInputTokens,
          cacheCreationInputTokens: usage.cacheCreationInputTokens,
          webSearchRequests: usage.webSearchRequests,
          costUSD: usage.costUSD,
        },
      ])
    ),
    lastSessionId: sessionId,
  }

  // 写入项目配置文件
  fs.writeFileSync('.claude/project.json', JSON.stringify(config, null, 2))
}

// === 3. 恢复会话成本 ===
function restoreSessionCosts(sessionId: string): boolean {
  const configPath = '.claude/project.json'
  if (!fs.existsSync(configPath)) return false

  const config: ProjectConfig = JSON.parse(fs.readFileSync(configPath, 'utf8'))

  // 仅恢复相同会话的成本
  if (config.lastSessionId !== sessionId) return false

  // 恢复状态
  state.totalCostUSD = config.lastCost ?? 0

  if (config.lastModelUsage) {
    state.modelUsage = Object.fromEntries(
      Object.entries(config.lastModelUsage).map(([model, usage]) => [
        model,
        { ...usage },
      ])
    )
  }

  return true
}

// === 4. 退出时自动保存 ===
process.on('exit', () => {
  saveSessionCosts(currentSessionId)
})
```

---

## 8. 关键代码路径总结

### 8.1 调用链路

```
API响应 → addToTotalSessionCost()
                ↓
        calculateUSDCost() → getModelCosts() → MODEL_COSTS[shortName]
                ↓                       ↓
        tokensToUSDCost()       getOpus46CostTier() (特殊处理)
                ↓
        addToTotalModelUsage() → STATE.modelUsage[model]
                ↓
        addToTotalCostState() → STATE.totalCostUSD += cost
                ↓
        遥测计数器更新 (costCounter, tokenCounter)
```

### 8.2 文件依赖图

```
commands/cost/cost.ts
    └── cost-tracker.ts
        ├── utils/modelCost.ts ← 核心计算逻辑
        │   └── utils/model/model.ts ← 模型名规范化
        │       └── utils/model/configs.ts ← 模型配置
        ├── bootstrap/state.ts ← 状态存储
        └── utils/advisor.ts ← Advisor成本处理
```

### 8.3 核心API速查

| 函数 | 文件 | 作用 |
|-----|------|------|
| `calculateUSDCost(model, usage)` | modelCost.ts | 计算单次成本 |
| `addToTotalSessionCost(cost, usage, model)` | cost-tracker.ts | 添加到会话统计 |
| `getTotalCostUSD()` | state.ts | 获取总成本 |
| `getModelUsage()` | state.ts | 获取模型统计 |
| `formatTotalCost()` | cost-tracker.ts | 格式化报告 |
| `getCanonicalName(model)` | model.ts | 规范化模型名 |

---

## 9. 特殊场景处理

### 9.1 未知模型定价

**文件位置**: `utils/modelCost.ts` 第89行、第166-173行

```typescript
// 默认未知模型定价
const DEFAULT_UNKNOWN_MODEL_COST = COST_TIER_5_25  // 使用$5/$25层

function trackUnknownModelCost(model: string, shortName: ModelShortName): void {
  // 记录分析事件
  logEvent('tengu_unknown_model_cost', {
    model: model,
    shortName: shortName,
  })
  // 设置警告标志
  setHasUnknownModelCost()
}

// 成本显示时添加警告
function formatTotalCost(): string {
  const costDisplay =
    formatCost(getTotalCostUSD()) +
    (hasUnknownModelCost()
      ? ' (costs may be inaccurate due to usage of unknown models)'
      : '')
  ...
}
```

### 9.2 Opus 4.6快速模式

**文件位置**: `utils/modelCost.ts` 第94-99行

```typescript
/**
 * Opus 4.6根据speed参数动态定价
 * 快速模式: $30/$150 (6x标准)
 * 标准模式: $5/$25
 */
export function getOpus46CostTier(fastMode: boolean): ModelCosts {
  if (isFastModeEnabled() && fastMode) {
    return COST_TIER_30_150
  }
  return COST_TIER_5_25
}
```

### 9.3 Advisor工具成本

**文件位置**: `cost-tracker.ts` 第304-322行

```typescript
// Advisor是后台运行的更强模型
// 其Token使用嵌套在主API响应的iterations字段中
for (const advisorUsage of getAdvisorUsage(usage)) {
  const advisorCost = calculateUSDCost(advisorUsage.model, advisorUsage)

  // 记录遥测事件
  logEvent('tengu_advisor_tool_token_usage', {
    advisor_model: advisorUsage.model,
    input_tokens: advisorUsage.input_tokens,
    output_tokens: advisorUsage.output_tokens,
    cost_usd_micros: Math.round(advisorCost * 1_000_000),
  })

  // 递归添加到会话成本
  totalCost += addToTotalSessionCost(advisorCost, advisorUsage, advisorUsage.model)
}
```

---

*API成本计算详解 - 完整技术文档*