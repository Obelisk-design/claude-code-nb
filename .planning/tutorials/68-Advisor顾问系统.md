# Advisor顾问系统 - 逐行细粒度分析

**文档创建日期:** 2026-04-02

---

## 目录

1. [Advisor概述](#1-advisor概述)
2. [commands/advisor.ts - 命令处理](#2-commandsadvisorts---命令处理)
3. [utils/advisor.ts - 核心工具函数](#3-utilsadvisorts---核心工具函数)
4. [顾问模型](#4-顾问模型)
5. [建议生成](#5-建议生成)
6. [用户使用指南](#6-用户使用指南)
7. [从零实现](#7-从零实现)

---

## 1. Advisor概述

### 1.1 什么是Advisor？

Advisor（顾问）是Claude Code CLI中的一个**智能审查系统**，它使用一个更强的模型来审查主模型的对话历史，提供改进建议和质量保证。这是一个服务端工具（server-side tool），由Anthropic API直接支持。

### 1.2 核心设计理念

```
主模型 (Sonnet/Opus) <---> 用户交互
         |
         v
    Advisor工具调用（自动转发完整对话历史）
         |
         v
审查模型 (Opus) <---> 分析并返回建议
         |
         v
主模型接收建议并继续工作
```

**关键特性:**
- **自动转发**：无需手动传参，整个对话历史自动发送给审查模型
- **服务端工具**：`server_tool_use` 类型，不是客户端工具
- **Beta功能**：需要 `advisor-tool-2026-03-01` beta header
- **实验性**：可通过GrowthBook特性开关控制

### 1.3 架构位置

```
用户命令 (/advisor opus)
    |
    v
commands/advisor.ts (命令处理)
    |
    v
state/AppStateStore.ts (状态存储: advisorModel)
    |
    v
main.tsx (启动时初始化)
    |
    v
query.ts (传递给API调用)
    |
    v
services/api/claude.ts (实际API集成)
    |
    v
components/messages/AdvisorMessage.tsx (UI渲染)
```

---

## 2. commands/advisor.ts - 命令处理

**文件路径:** `commands/advisor.ts`

### 2.1 导入分析

```typescript
import type { Command } from '../commands.js'
import type { LocalCommandCall } from '../types/command.js'
import {
  canUserConfigureAdvisor,
  isValidAdvisorModel,
  modelSupportsAdvisor,
} from '../utils/advisor.js'
import {
  getDefaultMainLoopModelSetting,
  normalizeModelStringForAPI,
  parseUserSpecifiedModel,
} from '../utils/model/model.js'
import { validateModel } from '../utils/model/validateModel.js'
import { updateSettingsForSource } from '../utils/settings/settings.js'
```

**逐行解释:**
- `Command` 类型：定义命令的基本结构
- `LocalCommandCall`：本地命令的调用签名
- `canUserConfigureAdvisor`：检查用户是否有权限配置advisor
- `isValidAdvisorModel`：验证模型是否可作为advisor
- `modelSupportsAdvisor`：检查基础模型是否支持advisor功能
- 模型工具函数：处理模型名称的解析和规范化
- `updateSettingsForSource`：持久化设置到用户配置文件

### 2.2 命令处理函数

```typescript
const call: LocalCommandCall = async (args, context) => {
  const arg = args.trim().toLowerCase()
  const baseModel = parseUserSpecifiedModel(
    context.getAppState().mainLoopModel ?? getDefaultMainLoopModelSetting(),
  )
```

**参数处理流程:**
1. `args.trim().toLowerCase()` - 规范化用户输入
2. 获取当前基础模型（主循环使用的模型）
3. 如果用户未设置模型，使用默认模型设置

### 2.3 无参数情况：显示当前状态

```typescript
if (!arg) {
  const current = context.getAppState().advisorModel
  if (!current) {
    return {
      type: 'text',
      value:
        'Advisor: not set\nUse "/advisor <model>" to enable (e.g. "/advisor opus").',
    }
  }
  if (!modelSupportsAdvisor(baseModel)) {
    return {
      type: 'text',
      value: `Advisor: ${current} (inactive)\nThe current model (${baseModel}) does not support advisors.`,
    }
  }
  return {
    type: 'text',
    value: `Advisor: ${current}\nUse "/advisor unset" to disable or "/advisor <model>" to change.`,
  }
}
```

**三种状态:**
1. **未设置**：提示如何启用
2. **已设置但不兼容**：显示 "(inactive)" 状态
3. **已设置且可用**：显示当前设置和操作选项

### 2.4 禁用Advisor

```typescript
if (arg === 'unset' || arg === 'off') {
  const prev = context.getAppState().advisorModel
  context.setAppState(s => {
    if (s.advisorModel === undefined) return s
    return { ...s, advisorModel: undefined }
  })
  updateSettingsForSource('userSettings', { advisorModel: undefined })
  return {
    type: 'text',
    value: prev
      ? `Advisor disabled (was ${prev}).`
      : 'Advisor already unset.',
  }
}
```

**关键操作:**
1. 获取之前的设置用于反馈
2. 更新内存状态（`setAppState`）
3. 持久化到用户设置文件（`updateSettingsForSource`）
4. 返回确认消息

### 2.5 设置Advisor模型

```typescript
const normalizedModel = normalizeModelStringForAPI(arg)
const resolvedModel = parseUserSpecifiedModel(arg)
const { valid, error } = await validateModel(resolvedModel)
if (!valid) {
  return {
    type: 'text',
    value: error
      ? `Invalid advisor model: ${error}`
      : `Unknown model: ${arg} (${resolvedModel})`,
  }
}
```

**验证流程:**
1. 规范化模型名称（处理别名）
2. 解析用户指定的模型字符串
3. 异步验证模型是否可用
4. 返回详细错误信息

### 2.6 双重验证机制

```typescript
if (!isValidAdvisorModel(resolvedModel)) {
  return {
    type: 'text',
    value: `The model ${arg} (${resolvedModel}) cannot be used as an advisor`,
  }
}
```

**两个不同的验证函数:**
- `validateModel()` - 检查模型是否存在和可用
- `isValidAdvisorModel()` - 检查模型是否具备advisor能力

### 2.7 保存设置

```typescript
context.setAppState(s => {
  if (s.advisorModel === normalizedModel) return s
  return { ...s, advisorModel: normalizedModel }
})
updateSettingsForSource('userSettings', { advisorModel: normalizedModel })
```

**性能优化:**
- 检查是否已设置相同值，避免不必要的状态更新

### 2.8 兼容性警告

```typescript
if (!modelSupportsAdvisor(baseModel)) {
  return {
    type: 'text',
    value: `Advisor set to ${normalizedModel}.\nNote: Your current model (${baseModel}) does not support advisors. Switch to a supported model to use the advisor.`,
  }
}
```

**用户体验:**
- 允许用户提前设置advisor
- 但明确告知当前模型不支持
- 建议切换到支持的模型

### 2.9 命令定义

```typescript
const advisor = {
  type: 'local',
  name: 'advisor',
  description: 'Configure the advisor model',
  argumentHint: '[<model>|off]',
  isEnabled: () => canUserConfigureAdvisor(),
  get isHidden() {
    return !canUserConfigureAdvisor()
  },
  supportsNonInteractive: true,
  load: () => Promise.resolve({ call }),
} satisfies Command
```

**命令配置详解:**
- `type: 'local'` - 本地命令，不需要API调用
- `argumentHint` - 提示参数格式
- `isEnabled()` - 动态检查功能是否可用
- `isHidden` - getter函数，动态隐藏（如果用户无权限）
- `supportsNonInteractive` - 支持非交互模式

---

## 3. utils/advisor.ts - 核心工具函数

**文件路径:** `utils/advisor.ts`

### 3.1 类型定义

#### 3.1.1 服务端工具调用块

```typescript
export type AdvisorServerToolUseBlock = {
  type: 'server_tool_use'
  id: string
  name: 'advisor'
  input: { [key: string]: unknown }
}
```

**关键字段:**
- `type: 'server_tool_use'` - 标识为服务端工具
- `name: 'advisor'` - 固定为'advisor'
- `input` - 通常为空对象（自动转发对话历史）

#### 3.1.2 工具结果块

```typescript
export type AdvisorToolResultBlock = {
  type: 'advisor_tool_result'
  tool_use_id: string
  content:
    | {
        type: 'advisor_result'
        text: string
      }
    | {
        type: 'advisor_redacted_result'
        encrypted_content: string
      }
    | {
        type: 'advisor_tool_result_error'
        error_code: string
      }
}
```

**三种结果类型:**
1. **advisor_result** - 明文建议
2. **advisor_redacted_result** - 加密结果（隐私保护）
3. **advisor_tool_result_error** - 错误情况

#### 3.1.3 联合类型

```typescript
export type AdvisorBlock = AdvisorServerToolUseBlock | AdvisorToolResultBlock
```

### 3.2 类型守卫

```typescript
export function isAdvisorBlock(param: {
  type: string
  name?: string
}): param is AdvisorBlock {
  return (
    param.type === 'advisor_tool_result' ||
    (param.type === 'server_tool_use' && param.name === 'advisor')
  )
}
```

**判断逻辑:**
- 如果是 `advisor_tool_result` 类型，一定是advisor块
- 如果是 `server_tool_use` 且名称为 'advisor'，也是advisor块

### 3.3 配置管理

```typescript
type AdvisorConfig = {
  enabled?: boolean
  canUserConfigure?: boolean
  baseModel?: string
  advisorModel?: string
}

function getAdvisorConfig(): AdvisorConfig {
  return getFeatureValue_CACHED_MAY_BE_STALE<AdvisorConfig>(
    'tengu_sage_compass',
    {},
  )
}
```

**GrowthBook特性标志:**
- `tengu_sage_compass` - advisor功能的特性标志名称
- 配置可以从远程动态更新
- 使用缓存值以提高性能

### 3.4 功能启用检查

```typescript
export function isAdvisorEnabled(): boolean {
  if (isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_ADVISOR_TOOL)) {
    return false
  }
  // The advisor beta header is first-party only (Bedrock/Vertex 400 on it).
  if (!shouldIncludeFirstPartyOnlyBetas()) {
    return false
  }
  return getAdvisorConfig().enabled ?? false
}
```

**三层检查:**
1. **环境变量覆盖** - `CLAUDE_CODE_DISABLE_ADVISOR_TOOL` 可强制禁用
2. **Beta头部限制** - 仅支持第一方API（不支持Bedrock/Vertex）
3. **特性开关** - GrowthBook配置的enabled字段

### 3.5 用户配置权限

```typescript
export function canUserConfigureAdvisor(): boolean {
  return isAdvisorEnabled() && (getAdvisorConfig().canUserConfigure ?? false)
}
```

**两个条件:**
- advisor功能本身启用
- 用户有配置权限（canUserConfigure）

### 3.6 实验模型配置

```typescript
export function getExperimentAdvisorModels():
  | { baseModel: string; advisorModel: string }
  | undefined {
  const config = getAdvisorConfig()
  return isAdvisorEnabled() &&
    !canUserConfigureAdvisor() &&
    config.baseModel &&
    config.advisorModel
    ? { baseModel: config.baseModel, advisorModel: config.advisorModel }
    : undefined
}
```

**用途:**
- 当用户不能自己配置时，可能由实验配置指定
- 返回固定的 baseModel + advisorModel 组合
- 用于A/B测试或渐进式发布

### 3.7 模型支持检查

```typescript
// @[MODEL LAUNCH]: Add the new model if it supports the advisor tool.
export function modelSupportsAdvisor(model: string): boolean {
  const m = model.toLowerCase()
  return (
    m.includes('opus-4-6') ||
    m.includes('sonnet-4-6') ||
    process.env.USER_TYPE === 'ant'
  )
}
```

**支持的模型:**
- Opus 4.6 系列
- Sonnet 4.6 系列
- 内部用户（USER_TYPE === 'ant'）可以使用所有模型

**注释提示:**
- `@[MODEL LAUNCH]` 是标记，提醒新模型发布时需要更新此列表

### 3.8 Advisor模型验证

```typescript
// @[MODEL LAUNCH]: Add the new model if it can serve as an advisor model.
export function isValidAdvisorModel(model: string): boolean {
  const m = model.toLowerCase()
  return (
    m.includes('opus-4-6') ||
    m.includes('sonnet-4-6') ||
    process.env.USER_TYPE === 'ant'
  )
}
```

**与 modelSupportsAdvisor 的区别:**
- `modelSupportsAdvisor` - 检查**基础模型**是否能调用advisor
- `isValidAdvisorModel` - 检查模型是否能**作为advisor**提供建议

### 3.9 初始设置获取

```typescript
export function getInitialAdvisorSetting(): string | undefined {
  if (!isAdvisorEnabled()) {
    return undefined
  }
  return getInitialSettings().advisorModel
}
```

**逻辑:**
- 功能未启用时返回undefined
- 否则从持久化设置中读取

### 3.10 使用量统计

```typescript
export function getAdvisorUsage(
  usage: BetaUsage,
): Array<BetaUsage & { model: string }> {
  const iterations = usage.iterations as
    | Array<{ type: string }>
    | null
    | undefined
  if (!iterations) {
    return []
  }
  return iterations.filter(
    it => it.type === 'advisor_message',
  ) as unknown as Array<BetaUsage & { model: string }>
}
```

**用途:**
- 从API响应的usage信息中提取advisor使用数据
- 用于成本计算和日志记录
- `iterations` 数组包含每次advisor调用的token统计

### 3.11 Advisor工具指令（核心）

```typescript
export const ADVISOR_TOOL_INSTRUCTIONS = `# Advisor Tool

You have access to an \`advisor\` tool backed by a stronger reviewer model. It takes NO parameters -- when you call it, your entire conversation history is automatically forwarded. The advisor sees the task, every tool call you've made, every result you've seen.

Call advisor BEFORE substantive work -- before writing code, before committing to an interpretation, before building on an assumption. If the task requires orientation first (finding files, reading code, seeing what's there), do that, then call advisor. Orientation is not substantive work. Writing, editing, and declaring an answer are.

Also call advisor:
- When you believe the task is complete. BEFORE this call, make your deliverable durable: write the file, stage the change, save the result. The advisor call takes time; if the session ends during it, a durable result persists and an unwritten one doesn't.
- When stuck -- errors recurring, approach not converging, results that don't fit.
- When considering a change of approach.

On tasks longer than a few steps, call advisor at least once before committing to an approach and once before declaring done. On short reactive tasks where the next action is dictated by tool output you just read, you don't need to keep calling -- the advisor adds most of its value on the first call, before the approach crystallizes.

Give the advice serious weight. If you follow a step and it fails empirically, or you have primary-source evidence that contradicts a specific claim (the file says X, the code does Y), adapt. A passing self-test is not evidence the advice is wrong -- it's evidence your test doesn't check what the advice is checking.

If you've already retrieved data pointing one way and the advisor points another: don't silently switch. Surface the conflict in one more advisor call -- "I found X, you suggest Y, which constraint breaks the tie?" The advisor saw your evidence but may have underweighted it; a reconcile call is cheaper than committing to the wrong branch.`
```

**指令解析:**

#### 核心概念
1. **无参数调用** - 自动转发完整对话历史
2. **更强模型** - 由更强大的模型审查
3. **全局可见** - advisor看到所有工具调用和结果

#### 调用时机
1. **实质性工作前** - 写代码、做判断、构建假设之前
2. **任务完成时** - 在advisor调用前确保持久化结果
3. **卡住时** - 错误反复、方法不收敛
4. **改变方法时** - 考虑转向其他方案

#### 最佳实践
- **先定向，后advisor** - 先找文件、读代码，再调用
- **长任务至少两次** - 开始前和结束前各一次
- **短任务可省略** - 反应式任务（根据工具输出决定下一步）不需要频繁调用

#### 权重处理
- **认真对待建议** - 给予足够重视
- **实证优先** - 如果实际操作失败，可以调整
- **冲突表面化** - 不要默默切换，明确指出冲突

---

## 4. 顾问模型

### 4.1 支持矩阵

| 模型类型 | 作为基础模型 | 作为Advisor |
|---------|------------|------------|
| Opus 4.6 | ✅ | ✅ |
| Sonnet 4.6 | ✅ | ✅ |
| 其他模型 | ❌ | ❌ |
| 内部用户(ant) | ✅ (所有) | ✅ (所有) |

### 4.2 模型选择策略

```typescript
// 在 services/api/claude.ts 中
let advisorModel: string | undefined
if (isAgenticQuery && isAdvisorEnabled()) {
  let advisorOption = options.advisorModel

  const advisorExperiment = getExperimentAdvisorModels()
  if (advisorExperiment !== undefined) {
    if (
      normalizeModelStringForAPI(advisorExperiment.baseModel) ===
      normalizeModelStringForAPI(options.model)
    ) {
      // 实验配置覆盖用户设置
      advisorOption = advisorExperiment.advisorModel
    }
  }

  if (advisorOption) {
    const normalizedAdvisorModel = normalizeModelStringForAPI(
      parseUserSpecifiedModel(advisorOption),
    )
    if (!modelSupportsAdvisor(options.model)) {
      logForDebugging(
        `[AdvisorTool] Skipping advisor - base model ${options.model} does not support advisor`,
      )
    } else if (!isValidAdvisorModel(normalizedAdvisorModel)) {
      logForDebugging(
        `[AdvisorTool] Skipping advisor - ${normalizedAdvisorModel} is not a valid advisor model`,
      )
    } else {
      advisorModel = normalizedAdvisorModel
      logForDebugging(
        `[AdvisorTool] Server-side tool enabled with ${advisorModel} as the advisor model`,
      )
    }
  }
}
```

**选择优先级:**
1. 实验配置（如果匹配baseModel）
2. 用户设置的advisorModel
3. 空值（禁用advisor）

### 4.3 Beta头部处理

```typescript
// constants/betas.ts
export const ADVISOR_BETA_HEADER = 'advisor-tool-2026-03-01'

// services/api/claude.ts
if (isAdvisorEnabled()) {
  betas.push(ADVISOR_BETA_HEADER)
}
```

**关键点:**
- Beta头部必须包含才能使用advisor
- 即使非代理查询也需要，用于解析已有的advisor块

---

## 5. 建议生成

### 5.1 API集成

#### 工具Schema定义

```typescript
const extraToolSchemas = [...(options.extraToolSchemas ?? [])]
if (advisorModel) {
  extraToolSchemas.push({
    type: 'advisor_20260301',
    name: 'advisor',
    model: advisorModel,
  } as unknown as BetaToolUnion)
}
const allTools = [...toolSchemas, ...extraToolSchemas]
```

**结构:**
- `type: 'advisor_20260301'` - 工具类型标识
- `name: 'advisor'` - 工具名称
- `model: advisorModel` - 指定审查模型

#### 系统提示注入

```typescript
const systemPrompt = compactAndJoin(
  [
    getCLISyspromptPrefix({
      isNonInteractive: options.isNonInteractiveSession,
      hasAppendSystemPrompt: options.hasAppendSystemPrompt,
    }),
    ...systemPrompt,
    ...(advisorModel ? [ADVISOR_TOOL_INSTRUCTIONS] : []),
    ...(injectChromeHere ? [CHROME_TOOL_SEARCH_INSTRUCTIONS] : []),
  ].filter(Boolean),
)
```

**注入位置:**
- 在基础系统提示之后
- 在其他工具指令之前
- 仅当advisor启用时注入

### 5.2 流式响应处理

#### 工具调用检测

```typescript
case 'server_tool_use':
  contentBlocks[part.index] = {
    ...part.content_block,
    input: '' as unknown as { [key: string]: unknown },
  }
  if ((part.content_block.name as string) === 'advisor') {
    isAdvisorInProgress = true
    logForDebugging(`[AdvisorTool] Advisor tool called`)
    logEvent('tengu_advisor_tool_call', {
      model: options.model,
      advisor_model: advisorModel ?? 'unknown',
    })
  }
  break
```

**关键状态:**
- `isAdvisorInProgress` - 标记advisor正在进行
- 用于中断检测和用户体验

#### 结果接收

```typescript
if ((part.content_block.type as string) === 'advisor_tool_result') {
  isAdvisorInProgress = false
  logForDebugging(`[AdvisorTool] Advisor tool result received`)
}
```

### 5.3 消息处理

#### Advisor块过滤

```typescript
// utils/messages.ts
export function stripAdvisorBlocks(
  messages: (UserMessage | AssistantMessage)[],
): (UserMessage | AssistantMessage)[] {
  let changed = false
  const result = messages.map(msg => {
    if (msg.type !== 'assistant') return msg
    const content = msg.message.content
    const filtered = content.filter(b => !isAdvisorBlock(b))
    if (filtered.length === content.length) return msg
    changed = true
    if (
      filtered.length === 0 &&
      content.some(b => isAdvisorBlock(b))
    ) {
      filtered.push({
        type: 'text' as const,
        text: '[Advisor response]',
        citations: [],
      })
    }
    return { ...msg, message: { ...msg.message, content: filtered } }
  })
  return changed ? result : messages
}
```

**过滤场景:**
- API不支持advisor beta时，必须移除advisor块
- 如果消息变空，插入占位文本

#### 错误追踪

```typescript
if ((content.type as string) === 'advisor_tool_result') {
  const result = content as {
    tool_use_id: string
    content: { type: string }
  }
  if (result.content.type === 'advisor_tool_result_error') {
    erroredToolUseIDs.add(result.tool_use_id)
  }
}
```

### 5.4 成本计算

```typescript
// cost-tracker.ts
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
  totalCost += addToTotalSessionCost(
    advisorCost,
    advisorUsage,
    advisorUsage.model,
  )
}
```

**成本组成:**
- 输入tokens
- 输出tokens
- 缓存读取tokens
- 缓存创建tokens

---

## 6. 用户使用指南

### 6.1 启用Advisor

```bash
# 设置Opus作为顾问
/advisor opus

# 使用完整模型名
/advisor claude-opus-4-6-20250401

# 查看当前状态
/advisor
```

### 6.2 禁用Advisor

```bash
# 禁用
/advisor unset

# 或
/advisor off
```

### 6.3 输出示例

**启用成功:**
```
Advisor set to claude-opus-4-6-20250401.
```

**查看状态:**
```
Advisor: claude-opus-4-6-20250401
Use "/advisor unset" to disable or "/advisor <model>" to change.
```

**不兼容警告:**
```
Advisor: claude-opus-4-6-20250401 (inactive)
The current model (claude-sonnet-4) does not support advisors.
```

**未设置:**
```
Advisor: not set
Use "/advisor <model>" to enable (e.g. "/advisor opus").
```

### 6.4 UI渲染

**调用中:**
```
⏸ Advising using claude-opus-4-6-20250401
```

**成功结果:**
```
✓ Advisor has reviewed the conversation and will apply the feedback [Ctrl+O to expand]
```

**错误:**
```
Advisor unavailable (error_code)
```

**加密结果:**
```
✓ Advisor has reviewed the conversation and will apply the feedback
```

### 6.5 最佳实践

#### 何时调用Advisor

✅ **应该调用:**
- 开始编写代码前
- 做出关键决策前
- 任务完成前（确保持久化）
- 遇到困难或卡住时
- 准备改变方法时

❌ **不需要调用:**
- 简单的文件读取
- 根据工具输出直接决定下一步
- 已经建立明确路径的重复性工作

#### 调用频率

- **长任务（>5步）**：至少2次（开始+结束）
- **中等任务（3-5步）**：1次（开始前）
- **短任务（<3步）**：可选

---

## 7. 从零实现

### 7.1 最小实现

```typescript
// 1. 定义类型
type AdvisorBlock =
  | { type: 'server_tool_use'; id: string; name: 'advisor'; input: {} }
  | { type: 'advisor_tool_result'; tool_use_id: string; content: unknown }

// 2. 类型守卫
function isAdvisorBlock(block: { type: string; name?: string }): block is AdvisorBlock {
  return block.type === 'advisor_tool_result' ||
         (block.type === 'server_tool_use' && block.name === 'advisor')
}

// 3. 启用检查
function isAdvisorEnabled(): boolean {
  return !process.env.DISABLE_ADVISOR && featureFlags.advisorEnabled
}

// 4. 模型验证
function isValidAdvisorModel(model: string): boolean {
  return model.includes('opus') || model.includes('sonnet')
}

// 5. 命令处理
async function handleAdvisorCommand(args: string, context: Context) {
  if (!args) return { type: 'text', value: `Current: ${context.advisorModel || 'not set'}` }

  if (args === 'unset' || args === 'off') {
    context.advisorModel = undefined
    return { type: 'text', value: 'Advisor disabled' }
  }

  if (!isValidAdvisorModel(args)) {
    return { type: 'text', value: 'Invalid advisor model' }
  }

  context.advisorModel = args
  return { type: 'text', value: `Advisor set to ${args}` }
}
```

### 7.2 API集成实现

```typescript
// 1. 构建工具schema
function buildAdvisorTool(advisorModel: string) {
  return {
    type: 'advisor_20260301',
    name: 'advisor',
    model: advisorModel
  }
}

// 2. 注入系统提示
function buildSystemPrompt(basePrompt: string[], advisorEnabled: boolean) {
  return [
    ...basePrompt,
    ...(advisorEnabled ? [ADVISOR_INSTRUCTIONS] : [])
  ]
}

// 3. 处理流式响应
function handleStreamEvent(event: StreamEvent) {
  if (event.type === 'server_tool_use' && event.name === 'advisor') {
    console.log('[Advisor] Tool called')
    return { advisorInProgress: true }
  }

  if (event.type === 'advisor_tool_result') {
    console.log('[Advisor] Result received')
    return { advisorInProgress: false }
  }
}

// 4. 过滤消息（当beta不支持时）
function filterAdvisorBlocks(messages: Message[]): Message[] {
  return messages.map(msg => ({
    ...msg,
    content: msg.content.filter(block => !isAdvisorBlock(block))
  }))
}
```

### 7.3 状态管理实现

```typescript
// AppState
interface AppState {
  advisorModel?: string
  // ...其他状态
}

// 更新状态
function setAdvisorModel(model: string | undefined) {
  appState.advisorModel = model
  persistToSettings('advisorModel', model)
}

// 读取设置
function loadAdvisorSettings() {
  const settings = loadUserSettings()
  if (isAdvisorEnabled()) {
    appState.advisorModel = settings.advisorModel
  }
}
```

### 7.4 成本追踪实现

```typescript
function trackAdvisorCost(usage: Usage) {
  const advisorIterations = usage.iterations?.filter(
    it => it.type === 'advisor_message'
  ) ?? []

  let totalCost = 0
  for (const iteration of advisorIterations) {
    const cost = calculateCost(iteration.model, {
      input: iteration.input_tokens,
      output: iteration.output_tokens,
      cacheRead: iteration.cache_read_input_tokens,
      cacheCreation: iteration.cache_creation_input_tokens
    })
    totalCost += cost

    analytics.track('advisor_usage', {
      model: iteration.model,
      cost,
      tokens: iteration.input_tokens + iteration.output_tokens
    })
  }

  return totalCost
}
```

### 7.5 完整集成示例

```typescript
// main.ts - 初始化
async function initializeAdvisor() {
  if (!isAdvisorEnabled()) return

  const advisorModel = getInitialAdvisorSetting()
  if (advisorModel && isValidAdvisorModel(advisorModel)) {
    appState.advisorModel = advisorModel
    console.log(`[Advisor] Initialized with ${advisorModel}`)
  }
}

// query.ts - 传递给API
function buildQueryOptions(state: AppState): QueryOptions {
  return {
    model: state.mainLoopModel,
    advisorModel: state.advisorModel,
    // ...其他选项
  }
}

// services/api/claude.ts - API调用
async function* streamClaudeResponse(options: QueryOptions) {
  const betas = ['advisor-tool-2026-03-01']

  const tools = [
    ...buildLocalTools(),
    ...(options.advisorModel ? [buildAdvisorTool(options.advisorModel)] : [])
  ]

  const systemPrompt = buildSystemPrompt(
    basePrompt,
    !!options.advisorModel
  )

  const stream = await client.messages.stream({
    model: options.model,
    tools,
    system: systemPrompt,
    betas,
    // ...其他参数
  })

  for await (const event of stream) {
    handleStreamEvent(event)
    yield event
  }
}
```

---

## 总结

### 关键文件清单

| 文件 | 作用 |
|------|------|
| `commands/advisor.ts` | 命令处理（/advisor命令） |
| `utils/advisor.ts` | 核心逻辑（类型、检查、指令） |
| `services/api/claude.ts` | API集成（工具schema、流式处理） |
| `components/messages/AdvisorMessage.tsx` | UI渲染 |
| `utils/messages.ts` | 消息处理（过滤、清理） |
| `state/AppStateStore.ts` | 状态管理 |
| `constants/betas.ts` | Beta头部定义 |
| `cost-tracker.ts` | 成本追踪 |
| `main.tsx` | 初始化 |
| `query.ts` | 选项传递 |

### 设计亮点

1. **无参数设计** - 自动转发对话历史，简化使用
2. **服务端工具** - 降低客户端复杂度
3. **特性开关** - 支持渐进式发布和A/B测试
4. **双模型验证** - 区分基础模型支持和顾问模型资格
5. **实验配置** - 支持强制指定模型组合
6. **成本透明** - 单独追踪advisor使用成本

### 扩展建议

如果要扩展advisor系统：
1. 添加新的支持模型需更新 `modelSupportsAdvisor` 和 `isValidAdvisorModel`
2. 支持多轮advisor对话可在 `ADVISOR_TOOL_INSTRUCTIONS` 中添加策略
3. 自定义advisor行为可通过修改系统提示实现
4. 添加advisor结果缓存可优化成本

---

*文档版本: 1.0*
*最后更新: 2026-04-02*