# 19. Cost成本追踪系统详解

**分析日期**: 2026-04-02

## 1. 概述

### 1.1 Cost命令的作用

`/cost` 命令用于显示当前会话的总成本和持续时间统计信息。它提供：

- **总成本**: 基于Token使用量的美元成本
- **API持续时间**: API调用的总时间
- **Wall持续时间**: 实际经过的时间
- **代码变更统计**: 添加和删除的代码行数
- **按模型分类的使用统计**: 每个模型的输入/输出Token、缓存使用情况

### 1.2 API成本计算原理

Claude Code的成本追踪系统基于以下原理：

1. **Token计数**: 追踪每次API调用的输入Token、输出Token、缓存读取Token、缓存创建Token
2. **价格配置**: 不同模型有不同的价格表（每百万Token的价格）
3. **实时累积**: 每次API调用后立即更新总成本
4. **会话持久化**: 会话结束时保存成本数据到项目配置

核心公式：
```
总成本 = (输入Token/1M × 输入价格)
       + (输出Token/1M × 输出价格)
       + (缓存读取Token/1M × 缓存读取价格)
       + (缓存创建Token/1M × 缓存创建价格)
       + (Web搜索次数 × 单次价格)
```

---

## 2. commands/cost/ 目录逐行分析

### 2.1 index.ts - 命令入口

**文件位置**: `commands/cost/index.ts`

```typescript
// 第1-7行: 导入依赖
import type { Command } from '../../commands.js'
import { isClaudeAISubscriber } from '../../utils/auth.js'

// 第8-21行: 命令定义
const cost = {
  type: 'local',                    // 本地命令，不需要API调用
  name: 'cost',                     // 命令名称
  description: 'Show the total cost and duration of the current session',
  get isHidden() {
    // Ant用户即使是订阅者也显示成本（他们需要看到成本分解）
    if (process.env.USER_TYPE === 'ant') {
      return false
    }
    // 订阅用户隐藏此命令（他们有无限额度）
    return isClaudeAISubscriber()
  },
  supportsNonInteractive: true,     // 支持非交互模式
  load: () => import('./cost.js'),  // 懒加载实现
} satisfies Command
```

**关键设计**：
- **懒加载**: 使用 `load: () => import('./cost.js')` 减少启动时间
- **动态隐藏**: 使用 getter 函数动态判断是否隐藏命令
- **Ant特权**: Ant用户（内部用户）即使是订阅者也能看到成本

### 2.2 cost.ts - 命令实现

**文件位置**: `commands/cost/cost.ts`

```typescript
// 第1-4行: 导入依赖
import { formatTotalCost } from '../../cost-tracker.js'
import { currentLimits } from '../../services/claudeAiLimits.js'
import type { LocalCommandCall } from '../../types/command.js'
import { isClaudeAISubscriber } from '../../utils/auth.js'

// 第6-24行: 命令实现
export const call: LocalCommandCall = async () => {
  // 检查用户是否为Claude AI订阅者
  if (isClaudeAISubscriber()) {
    let value: string

    // 检查是否正在使用超额额度
    if (currentLimits.isUsingOverage) {
      value =
        'You are currently using your overages to power your Claude Code usage...'
    } else {
      value =
        'You are currently using your subscription to power your Claude Code usage'
    }

    // Ant用户显示额外成本信息
    if (process.env.USER_TYPE === 'ant') {
      value += `\n\n[ANT-ONLY] Showing cost anyway:\n ${formatTotalCost()}`
    }

    return { type: 'text', value }
  }

  // 非订阅者：返回完整成本信息
  return { type: 'text', value: formatTotalCost() }
}
```

**执行流程**：
1. 检查用户订阅状态
2. 订阅用户：显示订阅状态消息
3. Ant用户：额外显示成本信息
4. 非订阅用户：显示完整成本明细

---

## 3. cost-tracker.ts 核心文件逐行分析

**文件位置**: `cost-tracker.ts`

### 3.1 类型定义

```typescript
// 第71-80行: 存储成本状态类型
type StoredCostState = {
  totalCostUSD: number                    // 总成本（美元）
  totalAPIDuration: number               // API总持续时间（毫秒）
  totalAPIDurationWithoutRetries: number // 不含重试的API持续时间
  totalToolDuration: number              // 工具执行总时间
  totalLinesAdded: number                // 添加的代码行数
  totalLinesRemoved: number              // 删除的代码行数
  lastDuration: number | undefined       // 上次持续时间
  modelUsage: { [modelName: string]: ModelUsage } | undefined // 模型使用映射
}
```

### 3.2 Usage类型说明

来自 `@anthropic-ai/sdk` 的 BetaUsage 类型包含：

```typescript
interface Usage {
  input_tokens: number                    // 输入Token数量
  output_tokens: number                   // 输出Token数量
  cache_read_input_tokens?: number        // 缓存读取Token数量
  cache_creation_input_tokens?: number    // 缓存创建Token数量
  server_tool_use?: {
    web_search_requests?: number          // Web搜索请求次数
  }
  speed?: 'fast' | 'standard'            // 速度模式（fast mode）
}
```

### 3.3 成本计算函数

#### getModelUsage() - 获取模型使用统计

**位置**: 第826-828行 (`bootstrap/state.ts`)

```typescript
export function getModelUsage(): { [modelName: string]: ModelUsage } {
  return STATE.modelUsage
}
```

**返回值**: 模型名称到使用统计的映射表

**数据结构**:
```typescript
{
  "claude-sonnet-4-20250514": {
    inputTokens: 50000,
    outputTokens: 12000,
    cacheReadInputTokens: 30000,
    cacheCreationInputTokens: 15000,
    webSearchRequests: 5,
    costUSD: 0.89,
    contextWindow: 200000,
    maxOutputTokens: 8192
  },
  "claude-opus-4-5-20250514": { ... }
}
```

#### getTotalCostUSD() - 获取总成本

**位置**: 第566-568行 (`bootstrap/state.ts`)

```typescript
export function getTotalCostUSD(): number {
  return STATE.totalCostUSD
}
```

**返回值**: 当前会话的累计总成本（美元）

#### getTotalAPIDuration() - 获取API总时间

**位置**: 第570-572行 (`bootstrap/state.ts`)

```typescript
export function getTotalAPIDuration(): number {
  return STATE.totalAPIDuration
}
```

**返回值**: API调用累计时间（毫秒）

### 3.4 成本累积函数

#### addToTotalSessionCost() - 添加会话成本

**位置**: 第278-323行

```typescript
export function addToTotalSessionCost(
  cost: number,      // 本次API调用的成本
  usage: Usage,     // Token使用量
  model: string,    // 模型名称
): number {
  // 第283-276行: 更新模型使用统计
  const modelUsage = addToTotalModelUsage(cost, usage, model)
  addToTotalCostState(cost, modelUsage, model)

  // 第286-301行: 记录遥测指标
  const attrs =
    isFastModeEnabled() && usage.speed === 'fast'
      ? { model, speed: 'fast' }
      : { model }

  getCostCounter()?.add(cost, attrs)
  getTokenCounter()?.add(usage.input_tokens, { ...attrs, type: 'input' })
  getTokenCounter()?.add(usage.output_tokens, { ...attrs, type: 'output' })
  getTokenCounter()?.add(usage.cache_read_input_tokens ?? 0, {
    ...attrs,
    type: 'cacheRead',
  })
  getTokenCounter()?.add(usage.cache_creation_input_tokens ?? 0, {
    ...attrs,
    type: 'cacheCreation',
  })

  // 第303-322行: 处理Advisor使用（可选的辅助模型调用）
  let totalCost = cost
  for (const advisorUsage of getAdvisorUsage(usage)) {
    const advisorCost = calculateUSDCost(advisorUsage.model, advisorUsage)
    logEvent('tengu_advisor_tool_token_usage', { ... })
    totalCost += addToTotalSessionCost(
      advisorCost,
      advisorUsage,
      advisorUsage.model,
    )
  }
  return totalCost
}
```

**关键步骤**：
1. 累积模型使用统计（Token计数）
2. 更新全局成本状态
3. 记录OpenTelemetry指标
4. 递归处理Advisor工具的使用统计

#### addToTotalModelUsage() - 更新模型使用统计

**位置**: 第250-276行

```typescript
function addToTotalModelUsage(
  cost: number,
  usage: Usage,
  model: string,
): ModelUsage {
  // 获取或初始化模型使用记录
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

  // 累加各项指标
  modelUsage.inputTokens += usage.input_tokens
  modelUsage.outputTokens += usage.output_tokens
  modelUsage.cacheReadInputTokens += usage.cache_read_input_tokens ?? 0
  modelUsage.cacheCreationInputTokens += usage.cache_creation_input_tokens ?? 0
  modelUsage.webSearchRequests +=
    usage.server_tool_use?.web_search_requests ?? 0
  modelUsage.costUSD += cost

  // 更新上下文窗口信息
  modelUsage.contextWindow = getContextWindowForModel(model, getSdkBetas())
  modelUsage.maxOutputTokens = getModelMaxOutputTokens(model).default

  return modelUsage
}
```

---

## 4. 成本计算逻辑

### 4.1 核心计算函数

**文件位置**: `utils/modelCost.ts`

#### calculateUSDCost() - 计算美元成本

**位置**: 第177-180行

```typescript
export function calculateUSDCost(resolvedModel: string, usage: Usage): number {
  const modelCosts = getModelCosts(resolvedModel, usage)
  return tokensToUSDCost(modelCosts, usage)
}
```

#### tokensToUSDCost() - Token转美元

**位置**: 第131-142行

```typescript
function tokensToUSDCost(modelCosts: ModelCosts, usage: Usage): number {
  return (
    // 输入Token成本（每百万Token的价格）
    (usage.input_tokens / 1_000_000) * modelCosts.inputTokens +

    // 输出Token成本
    (usage.output_tokens / 1_000_000) * modelCosts.outputTokens +

    // 缓存读取Token成本（折扣价格）
    ((usage.cache_read_input_tokens ?? 0) / 1_000_000) *
      modelCosts.promptCacheReadTokens +

    // 缓存创建Token成本
    ((usage.cache_creation_input_tokens ?? 0) / 1_000_000) *
      modelCosts.promptCacheWriteTokens +

    // Web搜索请求成本
    (usage.server_tool_use?.web_search_requests ?? 0) *
      modelCosts.webSearchRequests
  )
}
```

### 4.2 不同模型价格表

**位置**: `utils/modelCost.ts` 第26-89行

#### 价格层级定义

```typescript
// Sonnet系列价格: $3输入/$15输出每百万Token
export const COST_TIER_3_15 = {
  inputTokens: 3,              // $3/M input
  outputTokens: 15,             // $15/M output
  promptCacheWriteTokens: 3.75, // $3.75/M cache write
  promptCacheReadTokens: 0.3,   // $0.30/M cache read (90%折扣)
  webSearchRequests: 0.01,      // $0.01/request
} as const

// Opus 4/4.1价格: $15输入/$75输出每百万Token
export const COST_TIER_15_75 = {
  inputTokens: 15,
  outputTokens: 75,
  promptCacheWriteTokens: 18.75,
  promptCacheReadTokens: 1.5,
  webSearchRequests: 0.01,
} as const

// Opus 4.5价格: $5输入/$25输出每百万Token
export const COST_TIER_5_25 = {
  inputTokens: 5,
  outputTokens: 25,
  promptCacheWriteTokens: 6.25,
  promptCacheReadTokens: 0.5,
  webSearchRequests: 0.01,
} as const

// Fast Mode价格（Opus 4.6专属）: $30输入/$150输出
export const COST_TIER_30_150 = {
  inputTokens: 30,
  outputTokens: 150,
  promptCacheWriteTokens: 37.5,
  promptCacheReadTokens: 3,
  webSearchRequests: 0.01,
} as const

// Haiku 3.5价格: $0.80输入/$4输出
export const COST_HAIKU_35 = {
  inputTokens: 0.8,
  outputTokens: 4,
  promptCacheWriteTokens: 1,
  promptCacheReadTokens: 0.08,
  webSearchRequests: 0.01,
} as const

// Haiku 4.5价格: $1输入/$5输出
export const COST_HAIKU_45 = {
  inputTokens: 1,
  outputTokens: 5,
  promptCacheWriteTokens: 1.25,
  promptCacheReadTokens: 0.1,
  webSearchRequests: 0.01,
} as const
```

#### 模型价格映射

**位置**: 第104-126行

```typescript
export const MODEL_COSTS: Record<ModelShortName, ModelCosts> = {
  // Haiku系列
  'claude-3.5-haiku': COST_HAIKU_35,
  'claude-haiku-4.5': COST_HAIKU_45,

  // Sonnet系列
  'claude-3.5-sonnet': COST_TIER_3_15,
  'claude-3.7-sonnet': COST_TIER_3_15,
  'claude-sonnet-4': COST_TIER_3_15,
  'claude-sonnet-4.5': COST_TIER_3_15,
  'claude-sonnet-4.6': COST_TIER_3_15,

  // Opus系列
  'claude-opus-4': COST_TIER_15_75,
  'claude-opus-4.1': COST_TIER_15_75,
  'claude-opus-4.5': COST_TIER_5_25,
  'claude-opus-4.6': COST_TIER_5_25,
}
```

### 4.3 缓存Token折扣机制

**缓存优势**：
- **缓存写入**: 25%额外成本（首次写入提示缓存）
- **缓存读取**: 90%折扣（后续使用缓存）

**示例计算**：

假设使用 Sonnet 4.5：

```
场景1: 无缓存
- 输入Token: 100,000
- 成本: (100,000/1,000,000) × $3 = $0.30

场景2: 使用缓存（缓存命中）
- 缓存读取Token: 100,000
- 成本: (100,000/1,000,000) × $0.30 = $0.03
- 节省: $0.27 (90%折扣)

场景3: 首次创建缓存
- 缓存创建Token: 100,000
- 成本: (100,000/1,000,000) × $3.75 = $0.375
- 额外成本: $0.075 (25%溢价)
- 后续收益: 每次命中节省$0.27
```

### 4.4 Fast Mode定价

**位置**: 第94-99行

```typescript
export function getOpus46CostTier(fastMode: boolean): ModelCosts {
  if (isFastModeEnabled() && fastMode) {
    return COST_TIER_30_150  // 6倍价格提升
  }
  return COST_TIER_5_25
}
```

**Fast Mode特点**：
- 仅适用于 Opus 4.6
- 价格提升6倍（$5→$30输入，$25→$150输出）
- 需要API响应中 `usage.speed === 'fast'`
- 需要全局启用 Fast Mode

---

## 5. 成本显示UI

### 5.1 formatTotalCost() - 格式化总成本

**位置**: 第228-244行

```typescript
export function formatTotalCost(): string {
  // 格式化成本显示
  const costDisplay =
    formatCost(getTotalCostUSD()) +
    (hasUnknownModelCost()
      ? ' (costs may be inaccurate due to usage of unknown models)'
      : '')

  // 格式化模型使用明细
  const modelUsageDisplay = formatModelUsage()

  // 组装完整输出
  return chalk.dim(
    `Total cost:            ${costDisplay}\n` +
      `Total duration (API):  ${formatDuration(getTotalAPIDuration())}
Total duration (wall): ${formatDuration(getTotalDuration())}
Total code changes:    ${getTotalLinesAdded()} lines added, ${getTotalLinesRemoved()} lines removed
${modelUsageDisplay}`,
  )
}
```

**输出示例**：

```
Total cost:            $0.89
Total duration (API):  12.5s
Total duration (wall): 45.2s
Total code changes:    127 lines added, 34 lines removed

Usage by model:
  claude-sonnet-4.5:  45,231 input, 12,847 output, 30,000 cache read, 15,000 cache write ($0.72)
  claude-opus-4.5:    12,500 input, 3,200 output, 0 cache read, 0 cache write ($0.17)
```

### 5.2 formatModelUsage() - 格式化模型使用

**位置**: 第181-226行

```typescript
function formatModelUsage(): string {
  const modelUsageMap = getModelUsage()
  if (Object.keys(modelUsageMap).length === 0) {
    return 'Usage:                 0 input, 0 output, 0 cache read, 0 cache write'
  }

  // 按短名称聚合使用量
  const usageByShortName: { [shortName: string]: ModelUsage } = {}
  for (const [model, usage] of Object.entries(modelUsageMap)) {
    const shortName = getCanonicalName(model)
    if (!usageByShortName[shortName]) {
      usageByShortName[shortName] = {
        inputTokens: 0,
        outputTokens: 0,
        cacheReadInputTokens: 0,
        cacheCreationInputTokens: 0,
        webSearchRequests: 0,
        costUSD: 0,
        contextWindow: 0,
        maxOutputTokens: 0,
      }
    }
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

**关键功能**：
- 按模型短名称聚合（处理不同变体）
- 格式化数字（千位分隔符）
- 可选显示Web搜索次数
- 每个模型显示独立成本

### 5.3 formatCost() - 成本格式化

**位置**: 第177-179行

```typescript
function formatCost(cost: number, maxDecimalPlaces: number = 4): string {
  return `$${cost > 0.5 ? round(cost, 100).toFixed(2) : cost.toFixed(maxDecimalPlaces)}`
}
```

**格式化规则**：
- 成本 > $0.5：保留2位小数
- 成本 ≤ $0.5：保留4位小数（默认）
- 示例：`$1.23`, `$0.0456`, `$0.0001`

---

## 6. 相关Hooks

### 6.1 costHook.ts - 退出时成本显示

**文件位置**: `costHook.ts`

```typescript
import { useEffect } from 'react'
import { formatTotalCost, saveCurrentSessionCosts } from './cost-tracker.js'
import { hasConsoleBillingAccess } from './utils/billing.js'
import type { FpsMetrics } from './utils/fpsTracker.js'

export function useCostSummary(
  getFpsMetrics?: () => FpsMetrics | undefined,
): void {
  useEffect(() => {
    // 注册退出时的回调
    const f = () => {
      // 检查是否有控制台计费权限
      if (hasConsoleBillingAccess()) {
        process.stdout.write('\n' + formatTotalCost() + '\n')
      }

      // 保存当前会话成本到配置文件
      saveCurrentSessionCosts(getFpsMetrics?.())
    }

    // 监听进程退出事件
    process.on('exit', f)

    // 清理函数：移除监听器
    return () => {
      process.off('exit', f)
    }
  }, [])
}
```

**关键流程**：
1. 注册 `process.on('exit')` 钩子
2. 退出时检查计费权限
3. 有权限则打印成本摘要
4. 保存会话成本到项目配置

### 6.2 计费权限检查

**文件位置**: `utils/billing.ts`

```typescript
export function hasConsoleBillingAccess(): boolean {
  // 检查是否禁用成本警告
  if (isEnvTruthy(process.env.DISABLE_COST_WARNINGS)) {
    return false
  }

  // 订阅用户不显示成本
  const isSubscriber = isClaudeAISubscriber()
  if (isSubscriber) return false

  // 检查认证来源
  const authSource = getAuthTokenSource()
  const hasApiKey = getAnthropicApiKey() !== null

  // 未登录用户不显示成本
  if (!authSource.hasToken && !hasApiKey) {
    return false
  }

  // 检查组织和工作区角色
  const config = getGlobalConfig()
  const orgRole = config.oauthAccount?.organizationRole
  const workspaceRole = config.oauthAccount?.workspaceRole

  if (!orgRole || !workspaceRole) {
    return false
  }

  // 管理员或计费角色有权限
  return (
    ['admin', 'billing'].includes(orgRole) ||
    ['workspace_admin', 'workspace_billing'].includes(workspaceRole)
  )
}
```

**权限层级**：
1. 环境变量 `DISABLE_COST_WARNINGS=true` → 禁用
2. Claude AI订阅者 → 隐藏
3. 未认证用户 → 隐藏
4. 组织角色：admin/billing → 显示
5. 工作区角色：workspace_admin/workspace_billing → 显示

### 6.3 会话成本持久化

#### saveCurrentSessionCosts() - 保存会话成本

**位置**: 第143-175行

```typescript
export function saveCurrentSessionCosts(fpsMetrics?: FpsMetrics): void {
  saveCurrentProjectConfig(current => ({
    ...current,
    lastCost: getTotalCostUSD(),
    lastAPIDuration: getTotalAPIDuration(),
    lastAPIDurationWithoutRetries: getTotalAPIDurationWithoutRetries(),
    lastToolDuration: getTotalToolDuration(),
    lastDuration: getTotalDuration(),
    lastLinesAdded: getTotalLinesAdded(),
    lastLinesRemoved: getTotalLinesRemoved(),
    lastTotalInputTokens: getTotalInputTokens(),
    lastTotalOutputTokens: getTotalOutputTokens(),
    lastTotalCacheCreationInputTokens: getTotalCacheCreationInputTokens(),
    lastTotalCacheReadInputTokens: getTotalCacheReadInputTokens(),
    lastTotalWebSearchRequests: getTotalWebSearchRequests(),
    lastFpsAverage: fpsMetrics?.averageFps,
    lastFpsLow1Pct: fpsMetrics?.low1PctFps,
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
      ]),
    ),
    lastSessionId: getSessionId(),
  }))
}
```

**保存内容**：
- 总成本和持续时间
- 代码变更统计
- Token使用统计
- FPS性能指标
- 每个模型的详细使用情况
- 会话ID（用于恢复验证）

#### restoreCostStateForSession() - 恢复会话成本

**位置**: 第130-137行

```typescript
export function restoreCostStateForSession(sessionId: string): boolean {
  const data = getStoredSessionCosts(sessionId)
  if (!data) {
    return false
  }
  setCostStateForRestore(data)
  return true
}
```

**使用场景**：
- 会话恢复时加载之前的成本数据
- 避免跨会话成本累积错误
- 通过会话ID验证确保数据一致性

---

## 7. 从零实现指南

### 7.1 设计成本追踪系统

#### 核心设计原则

1. **单一数据源**: 使用全局状态管理成本数据
2. **实时更新**: 每次API调用后立即更新
3. **模型无关**: 支持动态添加新模型价格
4. **持久化**: 会话结束时保存数据

#### 系统架构

```
┌─────────────────────────────────────────────────────────┐
│                      全局状态 (State)                     │
│  - totalCostUSD: number                                 │
│  - modelUsage: { [modelName: string]: ModelUsage }      │
│  - totalAPIDuration: number                             │
└─────────────────────────────────────────────────────────┘
                            ↑
                            │ 更新
                            │
┌─────────────────────────────────────────────────────────┐
│              成本追踪器 (Cost Tracker)                    │
│  - addToTotalSessionCost(cost, usage, model)            │
│  - formatTotalCost()                                     │
│  - saveCurrentSessionCosts()                            │
└─────────────────────────────────────────────────────────┘
                            ↑
                            │ 调用
                            │
┌─────────────────────────────────────────────────────────┐
│                 API调用处理层                             │
│  - 接收API响应                                           │
│  - 提取usage字段                                         │
│  - 计算成本                                              │
│  - 更新全局状态                                          │
└─────────────────────────────────────────────────────────┘
```

### 7.2 实现Token计数

#### 步骤1: 定义使用类型

```typescript
// 定义模型使用统计类型
interface ModelUsage {
  inputTokens: number
  outputTokens: number
  cacheReadInputTokens: number
  cacheCreationInputTokens: number
  webSearchRequests: number
  costUSD: number
  contextWindow: number
  maxOutputTokens: number
}

// 定义全局状态
interface State {
  totalCostUSD: number
  modelUsage: { [modelName: string]: ModelUsage }
  totalAPIDuration: number
  // ... 其他状态
}
```

#### 步骤2: 实现状态管理

```typescript
// state.ts
let state: State = {
  totalCostUSD: 0,
  modelUsage: {},
  totalAPIDuration: 0,
  // ...
}

export function getTotalCostUSD(): number {
  return state.totalCostUSD
}

export function getModelUsage(): { [modelName: string]: ModelUsage } {
  return state.modelUsage
}

export function getUsageForModel(model: string): ModelUsage | undefined {
  return state.modelUsage[model]
}
```

#### 步骤3: 实现Token累积

```typescript
// cost-tracker.ts
function addToTotalModelUsage(
  cost: number,
  usage: Usage,
  model: string,
): ModelUsage {
  // 获取或创建模型使用记录
  const modelUsage = getUsageForModel(model) ?? createEmptyModelUsage()

  // 累加Token计数
  modelUsage.inputTokens += usage.input_tokens
  modelUsage.outputTokens += usage.output_tokens
  modelUsage.cacheReadInputTokens += usage.cache_read_input_tokens ?? 0
  modelUsage.cacheCreationInputTokens += usage.cache_creation_input_tokens ?? 0
  modelUsage.webSearchRequests += usage.server_tool_use?.web_search_requests ?? 0
  modelUsage.costUSD += cost

  return modelUsage
}

export function addToTotalSessionCost(
  cost: number,
  usage: Usage,
  model: string,
): number {
  const modelUsage = addToTotalModelUsage(cost, usage, model)

  // 更新全局状态
  STATE.modelUsage[model] = modelUsage
  STATE.totalCostUSD += cost

  // 记录遥测（可选）
  recordTelemetry(cost, usage, model)

  return cost
}
```

### 7.3 实现价格配置

#### 步骤1: 定义价格类型

```typescript
// modelCost.ts
interface ModelCosts {
  inputTokens: number           // 每百万Token的输入价格
  outputTokens: number          // 每百万Token的输出价格
  promptCacheWriteTokens: number // 缓存写入价格
  promptCacheReadTokens: number  // 缓存读取价格
  webSearchRequests: number      // Web搜索单次价格
}
```

#### 步骤2: 定义价格表

```typescript
// 定义价格层级
const COST_TIER_3_15 = {
  inputTokens: 3,
  outputTokens: 15,
  promptCacheWriteTokens: 3.75,
  promptCacheReadTokens: 0.3,
  webSearchRequests: 0.01,
}

// 映射模型到价格层级
const MODEL_COSTS: Record<string, ModelCosts> = {
  'claude-sonnet-4.5': COST_TIER_3_15,
  'claude-opus-4.5': COST_TIER_5_25,
  // ...
}
```

#### 步骤3: 实现成本计算

```typescript
function tokensToUSDCost(modelCosts: ModelCosts, usage: Usage): number {
  return (
    (usage.input_tokens / 1_000_000) * modelCosts.inputTokens +
    (usage.output_tokens / 1_000_000) * modelCosts.outputTokens +
    ((usage.cache_read_input_tokens ?? 0) / 1_000_000) *
      modelCosts.promptCacheReadTokens +
    ((usage.cache_creation_input_tokens ?? 0) / 1_000_000) *
      modelCosts.promptCacheWriteTokens +
    (usage.server_tool_use?.web_search_requests ?? 0) *
      modelCosts.webSearchRequests
  )
}

export function calculateUSDCost(model: string, usage: Usage): number {
  const modelCosts = MODEL_COSTS[model] ?? DEFAULT_MODEL_COSTS
  return tokensToUSDCost(modelCosts, usage)
}
```

### 7.4 完整实现示例

#### 最小可行实现

```typescript
// ===== 1. 状态管理 =====
// state.ts
interface Usage {
  input_tokens: number
  output_tokens: number
  cache_read_input_tokens?: number
  cache_creation_input_tokens?: number
}

interface ModelUsage {
  inputTokens: number
  outputTokens: number
  cacheReadInputTokens: number
  cacheCreationInputTokens: number
  costUSD: number
}

let totalCost = 0
const modelUsage: Record<string, ModelUsage> = {}

export function getTotalCost() { return totalCost }
export function getModelUsage() { return modelUsage }

// ===== 2. 价格配置 =====
// pricing.ts
const PRICING = {
  'claude-sonnet-4.5': {
    inputPerMillion: 3,
    outputPerMillion: 15,
    cacheReadPerMillion: 0.3,
    cacheWritePerMillion: 3.75,
  },
  // 添加更多模型...
}

// ===== 3. 成本计算 =====
// cost-calculator.ts
function calculateCost(model: string, usage: Usage): number {
  const price = PRICING[model]
  if (!price) throw new Error(`Unknown model: ${model}`)

  const inputCost = (usage.input_tokens / 1_000_000) * price.inputPerMillion
  const outputCost = (usage.output_tokens / 1_000_000) * price.outputPerMillion
  const cacheReadCost = ((usage.cache_read_input_tokens ?? 0) / 1_000_000) *
    price.cacheReadPerMillion
  const cacheWriteCost = ((usage.cache_creation_input_tokens ?? 0) / 1_000_000) *
    price.cacheWritePerMillion

  return inputCost + outputCost + cacheReadCost + cacheWriteCost
}

// ===== 4. 追踪API调用 =====
// tracker.ts
import { calculateCost } from './cost-calculator'
import { totalCost, modelUsage } from './state'

export function trackAPICall(model: string, usage: Usage): void {
  const cost = calculateCost(model, usage)

  // 更新总成本
  totalCost += cost

  // 更新模型使用统计
  if (!modelUsage[model]) {
    modelUsage[model] = {
      inputTokens: 0,
      outputTokens: 0,
      cacheReadInputTokens: 0,
      cacheCreationInputTokens: 0,
      costUSD: 0,
    }
  }

  modelUsage[model].inputTokens += usage.input_tokens
  modelUsage[model].outputTokens += usage.output_tokens
  modelUsage[model].cacheReadInputTokens += usage.cache_read_input_tokens ?? 0
  modelUsage[model].cacheCreationInputTokens += usage.cache_creation_input_tokens ?? 0
  modelUsage[model].costUSD += cost
}

// ===== 5. 格式化输出 =====
// formatter.ts
export function formatCostReport(): string {
  let report = `Total cost: $${totalCost.toFixed(4)}\n\n`
  report += 'Usage by model:\n'

  for (const [model, usage] of Object.entries(modelUsage)) {
    report += `  ${model}:\n`
    report += `    Input: ${usage.inputTokens.toLocaleString()} tokens\n`
    report += `    Output: ${usage.outputTokens.toLocaleString()} tokens\n`
    report += `    Cache read: ${usage.cacheReadInputTokens.toLocaleString()} tokens\n`
    report += `    Cache write: ${usage.cacheCreationInputTokens.toLocaleString()} tokens\n`
    report += `    Cost: $${usage.costUSD.toFixed(4)}\n`
  }

  return report
}

// ===== 6. 使用示例 =====
// main.ts
import { trackAPICall } from './tracker'
import { formatCostReport } from './formatter'

// 模拟API响应
const apiResponse = {
  model: 'claude-sonnet-4.5',
  usage: {
    input_tokens: 50000,
    output_tokens: 12000,
    cache_read_input_tokens: 30000,
    cache_creation_input_tokens: 0,
  }
}

// 追踪成本
trackAPICall(apiResponse.model, apiResponse.usage)

// 打印报告
console.log(formatCostReport())
```

### 7.5 高级特性实现

#### 动态模型价格

```typescript
// 支持Fast Mode等动态定价
function getModelCosts(model: string, usage: Usage): ModelCosts {
  const baseCosts = MODEL_COSTS[model]

  // 检查Fast Mode
  if (usage.speed === 'fast' && isFastModeEnabled()) {
    return {
      ...baseCosts,
      inputTokens: baseCosts.inputTokens * 6,
      outputTokens: baseCosts.outputTokens * 6,
    }
  }

  return baseCosts
}
```

#### 未知模型处理

```typescript
const unknownModels = new Set<string>()

function trackUnknownModel(model: string): void {
  if (!unknownModels.has(model)) {
    unknownModels.add(model)
    console.warn(`Unknown model: ${model}, using default pricing`)
  }
}

function getModelCosts(model: string): ModelCosts {
  const costs = MODEL_COSTS[model]
  if (!costs) {
    trackUnknownModel(model)
    return DEFAULT_MODEL_COSTS
  }
  return costs
}
```

#### 会话持久化

```typescript
// 保存到配置文件
function saveSessionCosts(): void {
  const config = loadConfig()
  config.lastSession = {
    id: getCurrentSessionId(),
    totalCost: getTotalCost(),
    modelUsage: getModelUsage(),
    timestamp: Date.now(),
  }
  saveConfig(config)
}

// 恢复会话
function restoreSession(sessionId: string): boolean {
  const config = loadConfig()
  if (config.lastSession?.id === sessionId) {
    STATE.totalCost = config.lastSession.totalCost
    STATE.modelUsage = config.lastSession.modelUsage
    return true
  }
  return false
}
```

---

## 8. 关键文件清单

| 文件路径 | 功能说明 |
|---------|---------|
| `commands/cost/index.ts` | Cost命令定义和元数据 |
| `commands/cost/cost.ts` | Cost命令实现逻辑 |
| `cost-tracker.ts` | 成本追踪核心逻辑 |
| `costHook.ts` | 退出时成本显示钩子 |
| `bootstrap/state.ts` | 全局状态管理 |
| `utils/modelCost.ts` | 模型价格配置和计算 |
| `utils/billing.ts` | 计费权限检查 |
| `entrypoints/sdk/coreSchemas.ts` | ModelUsage类型定义 |

---

## 9. 总结

Claude Code的成本追踪系统采用以下关键设计：

1. **全局状态管理**: 使用单一状态对象管理所有成本数据
2. **模型价格配置**: 灵活的价格表支持多模型和动态定价
3. **缓存Token优化**: 90%折扣鼓励使用Prompt Caching
4. **实时追踪**: 每次API调用后立即更新成本
5. **会话持久化**: 支持会话恢复和成本累计
6. **权限控制**: 根据用户角色显示不同信息

通过理解这套系统，你可以：
- 自定义模型价格配置
- 实现自己的成本追踪系统
- 扩展新的计费维度
- 优化缓存策略以降低成本