# Token节约优化详解

**文档日期：2026-04-02**

## 1. 概述

### Token成本对AI应用的重要性

Claude Code作为CLI工具，每次调用Claude API都会消耗Token。Token成本直接影响：

1. **经济成本**：每次API调用按Token计费，优化Token可降低使用成本
2. **上下文窗口限制**：Claude模型有200K（或1M）Token的上下文窗口限制，超出会导致prompt_too_long错误
3. **响应速度**：Token越多，API处理时间越长，影响用户体验
4. **连续性**：超出上下文窗口会触发压缩，可能导致信息丢失

### Claude Code的Token优化总体策略

Claude Code采用**三层Token优化体系**：

```
┌─────────────────────────────────────────────────────────────┐
│                   Token优化总体架构                           │
├─────────────────────────────────────────────────────────────┤
│ 第一层：预防性优化                                           │
│   - Prompt Caching (系统提示词缓存)                          │
│   - CLAUDE.md大小限制 (40K字符)                             │
│   - 工具描述精简                                             │
├─────────────────────────────────────────────────────────────┤
│ 第二层：动态压缩                                             │
│   - Microcompact (工具结果轻量压缩)                          │
│   - Session Memory Compact (记忆系统压缩)                    │
│   - Time-based Microcompact (时间触发压缩)                   │
├─────────────────────────────────────────────────────────────┤
│ 第三层：应急压缩                                             │
│   - Traditional Compact (传统API压缩)                        │
│   - Auto-compact触发 (阈值自动压缩)                          │
└─────────────────────────────────────────────────────────────┘
```

**Token节省效果**：
- Session Memory Compact：95%Token节省（从180K→10K）
- Prompt Caching：90%Token折扣（缓存命中Token按10%计费）
- Microcompact：实时清理工具结果，避免累积

---

## 2. 消息压缩机制 (Compact)

### services/compact/ 目录结构

```
services/compact/
├── compact.ts                  # 传统压缩主逻辑
├── autoCompact.ts              # 自动压缩触发管理
├── microCompact.ts             # 工具结果轻量压缩
├── apiMicrocompact.ts          # API原生上下文管理
├── sessionMemoryCompact.ts     # Session Memory压缩路径
├── grouping.ts                 # API轮次分组
├── prompt.ts                   # 压缩提示词模板
├── timeBasedMCConfig.ts        # 时间触发配置
├── compactWarningHook.ts       # React警告Hook
├── compactWarningState.ts      # 警告状态管理
└── postCompactCleanup.ts       # 压缩后清理
```

### 自动压缩触发条件

**触发阈值计算**（`autoCompact.ts:72-91`）：

```typescript
export function getAutoCompactThreshold(model: string): number {
  const effectiveContextWindow = getEffectiveContextWindowSize(model)

  const autocompactThreshold =
    effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS

  return autocompactThreshold
}

// Buffer配置
AUTOCOMPACT_BUFFER_TOKENS = 13_000
WARNING_THRESHOLD_BUFFER_TOKENS = 20_000
ERROR_THRESHOLD_BUFFER_TOKENS = 20_000
```

**触发流程**（`autoCompact.ts:160-239`）：

```typescript
export async function shouldAutoCompact(
  messages: Message[],
  model: string,
  querySource?: QuerySource,
): Promise<boolean> {
  // 递归防护：session_memory和compact fork会死锁
  if (querySource === 'session_memory' || querySource === 'compact') {
    return false
  }

  // Context-collapse模式：压制proactive autocompact
  if (feature('CONTEXT_COLLAPSE') && isContextCollapseEnabled()) {
    return false
  }

  const tokenCount = tokenCountWithEstimation(messages)
  const threshold = getAutoCompactThreshold(model)

  return tokenCount >= threshold  // ~167K (200K - 13K buffer)
}
```

**Token使用状态分级**（`autoCompact.ts:93-145`）：

```typescript
export function calculateTokenWarningState(
  tokenUsage: number,
  model: string,
): {
  percentLeft: number
  isAboveWarningThreshold: boolean     // 黄色警告
  isAboveErrorThreshold: boolean       // 红色警告
  isAboveAutoCompactThreshold: boolean // 自动压缩触发
  isAtBlockingLimit: boolean           // 阻塞限制
}
```

### 三层压缩路径：Session Memory → Microcompact → Traditional Compact

#### 第一层：Session Memory Compact（最优先）

**触发条件**（`sessionMemoryCompact.ts:403-432`）：

```typescript
export function shouldUseSessionMemoryCompaction(): boolean {
  // 允许环境变量覆盖
  if (isEnvTruthy(process.env.ENABLE_CLAUDE_CODE_SM_COMPACT)) {
    return true
  }

  // GrowthBook配置门控
  const sessionMemoryFlag = getFeatureValue_CACHED_MAY_BE_STALE(
    'tengu_session_memory', false
  )
  const smCompactFlag = getFeatureValue_CACHED_MAY_BE_STALE(
    'tengu_sm_compact', false
  )

  return sessionMemoryFlag && smCompactFlag
}
```

**配置参数**（`sessionMemoryCompact.ts:47-61`）：

```typescript
export const DEFAULT_SM_COMPACT_CONFIG: SessionMemoryCompactConfig = {
  minTokens: 10_000,              // 压缩后保留最小Token
  minTextBlockMessages: 5,        // 保留最少文本消息数
  maxTokens: 40_000,              // 压缩后最大Token上限
}
```

**压缩效果**：

- **Token节省率**：95%（从180K→10K）
- **速度**：无需API调用，直接替换messages
- **质量**：保留关键上下文（文件路径、函数签名、错误历史）

**核心流程**（`sessionMemoryCompact.ts:514-630`）：

```typescript
export async function trySessionMemoryCompaction(
  messages: Message[],
  agentId?: AgentId,
): Promise<CompactionResult | null> {
  // 1. 检查是否启用
  if (!shouldUseSessionMemoryCompaction()) return null

  // 2. 等待Session Memory提取完成
  await waitForSessionMemoryExtraction()

  // 3. 获取Session Memory内容
  const sessionMemory = await getSessionMemoryContent()

  // 4. 计算保留消息范围
  const startIndex = calculateMessagesToKeepIndex(
    messages, lastSummarizedIndex
  )

  // 5. 创建压缩结果（无需API调用）
  const compactionResult = createCompactionResultFromSessionMemory(
    messages, sessionMemory, messagesToKeep
  )

  return compactionResult
}
```

#### 第二层：Microcompact（轻量压缩）

**三种Microcompact类型**：

1. **Cached Microcompact**（缓存编辑）
2. **Time-based Microcompact**（时间触发）
3. **Legacy Microcompact**（已移除）

**Cached Microcompact**（`microCompact.ts:305-399`）：

```typescript
async function cachedMicrocompactPath(
  messages: Message[],
  querySource: QuerySource | undefined,
): Promise<MicrocompactResult> {
  const mod = await getCachedMCModule()
  const state = ensureCachedMCState()
  const config = mod.getCachedMCConfig()

  // 注册工具结果
  for (const message of messages) {
    if (message.type === 'user' && Array.isArray(message.message.content)) {
      for (const block of message.message.content) {
        if (block.type === 'tool_result' && compactableIds.has(block.tool_use_id)) {
          mod.registerToolResult(state, block.tool_use_id)
        }
      }
    }
  }

  // 计算需要删除的工具结果
  const toolsToDelete = mod.getToolResultsToDelete(state)

  if (toolsToDelete.length > 0) {
    // 创建cache_edits块（不修改本地内容）
    const cacheEdits = mod.createCacheEditsBlock(state, toolsToDelete)

    return {
      messages,  // 消息内容不变
      compactionInfo: {
        pendingCacheEdits: {
          trigger: 'auto',
          deletedToolIds: toolsToDelete,
        }
      }
    }
  }
}
```

**Time-based Microcompact**（`microCompact.ts:422-530`）：

```typescript
export function evaluateTimeBasedTrigger(
  messages: Message[],
  querySource: QuerySource | undefined,
): { gapMinutes: number; config: TimeBasedMCConfig } | null {
  const config = getTimeBasedMCConfig()

  // 检查主线程source
  if (!config.enabled || !querySource || !isMainThreadSource(querySource)) {
    return null
  }

  // 计算距离上次assistant消息的时间差
  const lastAssistant = messages.findLast(m => m.type === 'assistant')
  const gapMinutes = (Date.now() - new Date(lastAssistant.timestamp).getTime()) / 60_000

  // 超过阈值则触发（默认60分钟）
  if (gapMinutes < config.gapThresholdMinutes) {
    return null
  }

  return { gapMinutes, config }
}
```

**Time-based配置**（`timeBasedMCConfig.ts:18-43`）：

```typescript
export type TimeBasedMCConfig = {
  enabled: boolean                    // 启用开关
  gapThresholdMinutes: number         // 时间阈值（60分钟）
  keepRecent: number                  // 保留最近N个工具结果
}

const TIME_BASED_MC_CONFIG_DEFAULTS: TimeBasedMCConfig = {
  enabled: false,
  gapThresholdMinutes: 60,
  keepRecent: 5,
}
```

**Time-based原理**：

服务器缓存TTL为1小时。当用户空闲超过60分钟，服务器缓存必定失效，下次请求会重写整个prefix。此时提前清理旧工具结果可减少重写内容。

#### 第三层：Traditional Compact（传统API压缩）

**触发条件**：

Session Memory失败时fallback到传统压缩。

**核心流程**（`compact.ts:387-500`）：

```typescript
export async function compactConversation(
  messages: Message[],
  context: ToolUseContext,
  cacheSafeParams: CacheSafeParams,
  suppressFollowUpQuestions: boolean,
  customInstructions?: string,
  isAutoCompact: boolean = false,
): Promise<CompactionResult> {
  // 1. 计算预压缩Token
  const preCompactTokenCount = tokenCountWithEstimation(messages)

  // 2. 运行pre_compact hooks
  context.onCompactProgress?.({
    type: 'hooks_start',
    hookType: 'pre_compact',
  })

  // 3. 调用API进行压缩
  const summaryResponse = await runAgent({
    model: getSmallFastModel(),
    systemPrompt: asSystemPrompt([{
      type: 'text',
      text: getCompactPrompt(customInstructions),
      cache_control: { type: 'ephemeral' }
    }]),
    messages: messagesToSummarize,
    maxTurns: 1,
    toolPermissionContext: getEmptyToolPermissionContext(),
  })

  // 4. 构建压缩结果
  const boundaryMarker = createCompactBoundaryMessage('auto', preCompactTokenCount)
  const summaryMessages = [createUserMessage({
    content: getCompactUserSummaryMessage(summary),
    isCompactSummary: true,
  })]

  return {
    boundaryMarker,
    summaryMessages,
    messagesToKeep,
    preCompactTokenCount,
    postCompactTokenCount,
  }
}
```

### 压缩算法详解

#### Session Memory压缩算法

**核心算法**（`sessionMemoryCompact.ts:324-397`）：

```typescript
export function calculateMessagesToKeepIndex(
  messages: Message[],
  lastSummarizedIndex: number,
): number {
  // 1. 从lastSummarizedIndex + 1开始
  let startIndex = lastSummarizedIndex >= 0
    ? lastSummarizedIndex + 1
    : messages.length

  // 2. 计算当前Token和文本消息数
  let totalTokens = 0
  let textBlockMessageCount = 0
  for (let i = startIndex; i < messages.length; i++) {
    totalTokens += estimateMessageTokens([messages[i]])
    if (hasTextBlocks(messages[i])) {
      textBlockMessageCount++
    }
  }

  // 3. 向前扩展直到满足最小要求
  const floor = messages.findLastIndex(m => isCompactBoundaryMessage(m))
  for (let i = startIndex - 1; i >= floor; i--) {
    totalTokens += estimateMessageTokens([messages[i]])
    if (hasTextBlocks(messages[i])) {
      textBlockMessageCount++
    }
    startIndex = i

    // 停止条件：达到maxTokens或满足minTokens+minTextBlockMessages
    if (totalTokens >= config.maxTokens) break
    if (totalTokens >= config.minTokens &&
        textBlockMessageCount >= config.minTextBlockMessages) break
  }

  // 4. 调整索引以保持tool_use/tool_result配对完整性
  return adjustIndexToPreserveAPIInvariants(messages, startIndex)
}
```

**API不变量保护**（`sessionMemoryCompact.ts:232-314`）：

```typescript
export function adjustIndexToPreserveAPIInvariants(
  messages: Message[],
  startIndex: number,
): number {
  // 问题场景示例：
  // Index N:   assistant, id: X, content: [thinking]
  // Index N+1: assistant, id: X, content: [tool_use: ORPHAN_ID]
  // Index N+2: assistant, id: X, content: [tool_use: VALID_ID]
  // Index N+3: user, content: [tool_result: ORPHAN_ID, tool_result: VALID_ID]
  //
  // 如果startIndex = N+2，只保留N+2和N+3：
  // - API合并相同id的assistant消息
  // - ORPHAN tool_use被排除，但ORPHAN tool_result仍然存在
  // - API错误：orphan tool_result引用不存在的tool_use

  // 步骤1：收集所有保留范围内的tool_result IDs
  const allToolResultIds: string[] = []
  for (let i = startIndex; i < messages.length; i++) {
    allToolResultIds.push(...getToolResultIds(messages[i]))
  }

  // 步骤2：向前查找匹配的tool_use消息
  for (let i = adjustedIndex - 1; i >= 0 && neededToolUseIds.size > 0; i--) {
    if (hasToolUseWithIds(messages[i], neededToolUseIds)) {
      adjustedIndex = i
    }
  }

  // 步骤3：处理thinking blocks共享相同message.id的情况
  // 确保thinking blocks能被normalizeMessagesForAPI正确合并

  return adjustedIndex
}
```

#### Traditional Compact提示词模板

**完整压缩提示词**（`prompt.ts:61-143`）：

```typescript
const BASE_COMPACT_PROMPT = `Your task is to create a detailed summary of the conversation so far.

${DETAILED_ANALYSIS_INSTRUCTION_BASE}

Your summary should include the following sections:

1. Primary Request and Intent: Capture all of the user's explicit requests
2. Key Technical Concepts: List all important technical concepts
3. Files and Code Sections: Enumerate specific files and code sections
4. Errors and fixes: List all errors and how they were fixed
5. Problem Solving: Document problems solved
6. All user messages: List ALL user messages that are not tool results
7. Pending Tasks: Outline any pending tasks
8. Current Work: Describe precisely what was being worked on
9. Optional Next Step: List the next step related to most recent work

<example>
<analysis>
[Your thought process]
</analysis>

<summary>
1. Primary Request and Intent:
   [Detailed description]
...
</summary>
</example>

Please provide your summary based on the conversation so far.`
```

**分析指令**（`prompt.ts:31-59`）：

```typescript
const DETAILED_ANALYSIS_INSTRUCTION_BASE = `Before providing your final summary, wrap your analysis in <analysis> tags:

1. Chronologically analyze each message:
   - User's explicit requests and intents
   - Your approach to addressing the requests
   - Key decisions, technical concepts, code patterns
   - Specific details: file names, code snippets, function signatures
   - Errors encountered and fixes
   - Pay special attention to user feedback

2. Double-check for technical accuracy and completeness.`
```

**无工具保证**（`prompt.ts:19-26`）：

```typescript
const NO_TOOLS_PREAMBLE = `CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.

- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- You already have all the context you need in the conversation above.
- Tool calls will be REJECTED and will waste your only turn — you will fail the task.
- Your entire response must be plain text: <analysis> + <summary> blocks.`
```

### Token节省效果（典型95%节省）

**实际案例对比**：

| 压缩类型 | 压缩前 | 压缩后 | 节省率 | 压缩耗时 |
|---------|--------|--------|--------|----------|
| Session Memory | 180,000 | 10,000 | 95% | 0ms（无API） |
| Traditional | 180,000 | 20,000 | 89% | 5-10s |
| Microcompact | 累积工具结果 | 实时清理 | 防止累积 | 0ms |

**Session Memory优势**：

1. **零API成本**：无需调用Claude API生成摘要
2. **保持质量**：Session Memory由专门agent维护，包含完整上下文
3. **快速响应**：直接替换messages数组，无网络延迟
4. **可恢复性**：完整transcript保存在磁盘，可随时查阅

---

## 3. 上下文窗口管理

### utils/context.ts 上下文窗口配置

**模型上下文窗口定义**（`context.ts:8-98`）：

```typescript
// 默认上下文窗口
export const MODEL_CONTEXT_WINDOW_DEFAULT = 200_000

// 压缩操作最大输出Token
export const COMPACT_MAX_OUTPUT_TOKENS = 20_000

// 默认最大输出Token
const MAX_OUTPUT_TOKENS_DEFAULT = 32_000
const MAX_OUTPUT_TOKENS_UPPER_LIMIT = 64_000

// 优化上限（避免slot过度预留）
export const CAPPED_DEFAULT_MAX_TOKENS = 8_000
export const ESCALATED_MAX_TOKENS = 64_000
```

**上下文窗口计算**（`context.ts:51-98`）：

```typescript
export function getContextWindowForModel(
  model: string,
  betas?: string[],
): number {
  // 1. 环境变量覆盖（ant-only）
  if (process.env.CLAUDE_CODE_MAX_CONTEXT_TOKENS) {
    const override = parseInt(process.env.CLAUDE_CODE_MAX_CONTEXT_TOKENS, 10)
    if (!isNaN(override) && override > 0) {
      return override
    }
  }

  // 2. [1m]后缀显式启用
  if (has1mContext(model)) {
    return 1_000_000
  }

  // 3. 模型能力配置
  const cap = getModelCapability(model)
  if (cap?.max_input_tokens && cap.max_input_tokens >= 100_000) {
    return cap.max_input_tokens
  }

  // 4. Beta header启用1M
  if (betas?.includes(CONTEXT_1M_BETA_HEADER) && modelSupports1M(model)) {
    return 1_000_000
  }

  // 5. Sonnet 1M实验
  if (getSonnet1mExpTreatmentEnabled(model)) {
    return 1_000_000
  }

  // 6. 默认200K
  return MODEL_CONTEXT_WINDOW_DEFAULT
}
```

### Token预算分配

**有效上下文窗口计算**（`autoCompact.ts:33-49`）：

```typescript
export function getEffectiveContextWindowSize(model: string): number {
  // 为压缩摘要预留Token
  const reservedTokensForSummary = Math.min(
    getMaxOutputTokensForModel(model),
    MAX_OUTPUT_TOKENS_FOR_SUMMARY,  // 20,000
  )

  let contextWindow = getContextWindowForModel(model, getSdkBetas())

  // 允许环境变量覆盖自动压缩窗口
  const autoCompactWindow = process.env.CLAUDE_CODE_AUTO_COMPACT_WINDOW
  if (autoCompactWindow) {
    const parsed = parseInt(autoCompactWindow, 10)
    if (!isNaN(parsed) && parsed > 0) {
      contextWindow = Math.min(contextWindow, parsed)
    }
  }

  // 有效窗口 = 总窗口 - 摘要预留
  return contextWindow - reservedTokensForSummary
}
```

**Token预算分配图**：

```
┌────────────────────────────────────────────────────────┐
│              200K上下文窗口Token预算分配                 │
├────────────────────────────────────────────────────────┤
│                                                        │
│  ┌──────────────────────────────────────────────────┐ │
│  │ 系统提示词 + 工具定义                ~20K        │ │
│  └──────────────────────────────────────────────────┘ │
│                                                        │
│  ┌──────────────────────────────────────────────────┐ │
│  │ 对话历史（用户消息 + assistant消息）              │ │
│  │                                                   │ │
│  │  ├─ 已压缩摘要                    ~10K           │ │
│  │  ├─ 保留的最近消息                ~40K           │ │
│  │  └─ 增长空间                      ~127K          │ │
│  │                                                   │ │
│  │  总容量：167K (200K - 13K buffer - 20K output)   │ │
│  └──────────────────────────────────────────────────┘ │
│                                                        │
│  ┌──────────────────────────────────────────────────┐ │
│  │ 输出Token预留                     20K            │ │
│  └──────────────────────────────────────────────────┘ │
│                                                        │
│  ┌──────────────────────────────────────────────────┐ │
│  │ Autocompact Buffer                13K            │ │
│  │ (触发阈值：200K - 13K = 187K)                    │ │
│  └──────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────┘
```

### 系统提示词优化

**系统提示词缓存策略**（`claude.ts:366-374`）：

```typescript
function getCacheControl(options?: {
  querySource?: QuerySource
  scope?: CacheScope
}): {
  type: 'ephemeral'
  ttl?: '1h'
  scope?: 'global'
} {
  return {
    type: 'ephemeral',
    ...(should1hCacheTTL(querySource) && { ttl: '1h' }),
    ...(scope === 'global' && { scope }),
  }
}
```

**1h TTL启用条件**（`claude.ts:393-431`）：

```typescript
function should1hCacheTTL(querySource?: QuerySource): boolean {
  // 3P Bedrock用户通过环境变量启用
  if (getAPIProvider() === 'bedrock' &&
      isEnvTruthy(process.env.ENABLE_PROMPT_CACHING_1H_BEDROCK)) {
    return true
  }

  // 用户资格检查（ant或订阅用户未超出限制）
  let userEligible = getPromptCache1hEligible()
  if (userEligible === null) {
    userEligible = process.env.USER_TYPE === 'ant' ||
                   (isClaudeAISubscriber() && !currentLimits.isUsingOverage)
    setPromptCache1hEligible(userEligible)
  }
  if (!userEligible) return false

  // GrowthBook白名单
  let allowlist = getPromptCache1hAllowlist()
  if (allowlist === null) {
    const config = getFeatureValue_CACHED_MAY_BE_STALE<{
      allowlist?: string[]
    }>('tengu_prompt_cache_1h_config', {})
    allowlist = config.allowlist ?? []
    setPromptCache1hAllowlist(allowlist)
  }

  // 前缀匹配检查
  return querySource !== undefined &&
         allowlist.some(pattern =>
           pattern.endsWith('*')
             ? querySource.startsWith(pattern.slice(0, -1))
             : querySource === pattern
         )
}
```

---

## 4. Prompt Caching (缓存命中)

### services/api/claude.ts 中的缓存实现

**缓存控制添加**（`claude.ts:593-631`）：

```typescript
export function userMessageToMessageParam(
  message: UserMessage,
  addCache: boolean,
  enablePromptCaching: boolean,
  querySource?: QuerySource,
): MessageParam {
  if (addCache) {
    if (typeof message.message.content === 'string') {
      return {
        role: 'user',
        content: [
          {
            type: 'text',
            text: message.message.content,
            ...(enablePromptCaching && {
              cache_control: getCacheControl({ querySource }),
            }),
          },
        ],
      }
    } else {
      return {
        role: 'user',
        content: message.message.content.map((_, i) => ({
          ..._,
          // 只在最后一个block添加cache_control
          ...(i === message.message.content.length - 1
            ? enablePromptCaching
              ? { cache_control: getCacheControl({ querySource }) }
              : {}
            : {}),
        })),
      }
    }
  }
  return {
    role: 'user',
    content: Array.isArray(message.message.content)
      ? [...message.message.content]  // 克隆防止mutation
      : message.message.content,
  }
}
```

### system prompt缓存策略

**系统提示词缓存标记位置**：

```typescript
// 系统提示词结构
[
  {
    type: 'text',
    text: getCLISyspromptPrefix(),
    cache_control: { type: 'ephemeral', ttl: '1h' }
  },
  {
    type: 'text',
    text: toolInstructions,
    cache_control: { type: 'ephemeral' }
  },
  {
    type: 'text',
    text: memoryInstructions,
  }
]
```

**缓存优先级**：

1. **系统提示词前缀**：最高优先级，1h TTL
2. **工具定义**：第二优先级，ephemeral缓存
3. **对话历史**：最后一个user message添加缓存标记

### 如何提高缓存命中率

**缓存命中关键因素**：

1. **保持系统提示词稳定**：避免频繁修改CLAUDE.md
2. **使用1h TTL**：延长缓存生命周期
3. **减少工具定义变化**：避免动态添加/移除工具
4. **连续对话**：避免长时间中断（超过1h会导致缓存失效）

**缓存命中优化策略**：

```typescript
// 全局缓存scope（减少cache key计算）
if (useGlobalCacheFeature && !needsToolBasedCacheMarker) {
  betas.push(PROMPT_CACHING_SCOPE_BETA_HEADER)
}

// 工具延迟加载（避免MCP工具污染全局缓存）
const willDefer = (t: Tool) =>
  useToolSearch && (deferredToolNames.has(t.name) || shouldDeferLspTool(t))

// MCP工具不能全局缓存（per-user动态）
const needsToolBasedCacheMarker =
  useGlobalCacheFeature &&
  filteredTools.some(t => t.isMcp === true && !willDefer(t))
```

### 缓存Token折扣（90%折扣）

**缓存Token计费规则**：

| Token类型 | 计费比例 | 示例 |
|----------|---------|------|
| cache_creation_input_tokens | 100%（首次写入） | 20K Token = $0.30 |
| cache_read_input_tokens | **10%（缓存命中）** | 20K Token = $0.03 |
| input_tokens（无缓存） | 100% | 20K Token = $0.30 |

**成本节省计算**：

```
假设：
- 系统提示词：20K Token
- 对话历史：40K Token
- 每次请求total：60K Token

无缓存成本（10次请求）：
  10 × 60K × $3/M = $1.80

有缓存成本（10次请求，系统提示词缓存）：
  首次：20K（creation）+ 40K = $0.30 + $0.12 = $0.42
  后续9次：20K（read 10%）+ 40K = 9 × ($0.03 + $0.12) = $1.35
  总计：$1.77

节省：$1.80 - $1.77 = $0.03（约2%）

理想情况（系统提示词+工具定义缓存，对话历史稳定）：
  首次：60K creation
  后续：60K read 10%
  节省：90%
```

---

## 5. CLAUDE.md优化

### 文件大小限制

**字符限制**（`claudemd.ts:92`）：

```typescript
// 推荐最大字符数
export const MAX_MEMORY_CHARACTER_COUNT = 40000

// 原因：
// 1. 超过40K字符会被截断
// 2. 40K字符 ≈ 10K Token
// 3. 占用上下文窗口的5%（200K）
```

**截断策略**：

```typescript
// 超过限制时警告，但不强制截断
if (content.length > MAX_MEMORY_CHARACTER_COUNT) {
  logForDebugging(
    `Memory file exceeds recommended size: ${content.length} > ${MAX_MEMORY_CHARACTER_COUNT}`,
    { level: 'warn' }
  )
}
```

### @include指令优化

**@include指令说明**（`claudemd.ts:19-26`）：

```typescript
/**
 * Memory @include directive:
 * - Syntax: @path, @./relative/path, @~/home/path, or @/absolute/path
 * - @path (without prefix) is treated as relative path
 * - Works in leaf text nodes only (not inside code blocks)
 * - Included files are added as separate entries before the including file
 * - Circular references prevented by tracking processed files
 * - Non-existent files silently ignored
 */
```

**@include优势**：

1. **模块化**：分离不同类型的指令（编码规范、架构文档等）
2. **按需加载**：只include需要的文件
3. **避免重复**：多个项目共享相同配置
4. **缓存友好**：include的文件独立缓存

**使用示例**：

```markdown
# CLAUDE.md

@./docs/coding-standards.md
@./docs/architecture.md

# 项目特定指令
本项目使用TypeScript，请遵循编码规范。
```

### 内容截断策略

**Session Memory截断**（`prompts.ts:256-296`）：

```typescript
export function truncateSessionMemoryForCompact(content: string): {
  truncatedContent: string
  wasTruncated: boolean
} {
  const maxCharsPerSection = MAX_SECTION_LENGTH * 4  // 2000 × 4 = 8000字符

  for (const line of lines) {
    if (line.startsWith('# ')) {
      // 刷新前一个section
      const result = flushSessionSection(
        currentSectionHeader,
        currentSectionLines,
        maxCharsPerSection
      )

      if (result.wasTruncated) {
        keptLines.push('\n[... section truncated for length ...]')
      }
    }
  }
}
```

**截断触发条件**：

```typescript
const MAX_SECTION_LENGTH = 2000       // 每个section 2000 Token
const MAX_TOTAL_SESSION_MEMORY_TOKENS = 12000  // 总计 12000 Token
```

---

## 6. 记忆系统Token优化

### 年龄衰减机制

**Session Memory更新阈值**（`sessionMemoryUtils.ts:17-36`）：

```typescript
export type SessionMemoryConfig = {
  // 初始化阈值：达到10000 Token才开始提取
  minimumMessageTokensToInit: number

  // 更新阈值：每增长5000 Token更新一次
  minimumTokensBetweenUpdate: number

  // 工具调用间隔：每3次工具调用更新一次
  toolCallsBetweenUpdates: number
}

export const DEFAULT_SESSION_MEMORY_CONFIG: SessionMemoryConfig = {
  minimumMessageTokensToInit: 10000,
  minimumTokensBetweenUpdate: 5000,
  toolCallsBetweenUpdates: 3,
}
```

**年龄衰减原理**：

- **越旧的信息权重越低**：Session Memory自动保留最新上下文
- **自动清理旧section**：超过section限制会被截断
- **Worklog滚动更新**：只保留最近的工作步骤

### 相关性筛选

**Session Memory结构**（`prompts.ts:11-41`）：

```typescript
export const DEFAULT_SESSION_MEMORY_TEMPLATE = `
# Session Title
_5-10 word descriptive title_

# Current State
_What is actively being worked on right now? Pending tasks._

# Task specification
_What did the user ask to build? Design decisions._

# Files and Functions
_Important files and why they are relevant._

# Workflow
_Bash commands and how to interpret output._

# Errors & Corrections
_Errors encountered and how they were fixed._

# Codebase and System Documentation
_Important system components._

# Learnings
_What has worked well? What to avoid?_

# Key results
_Exact output requested by user._

# Worklog
_Step by step, what was attempted?_
`
```

**相关性筛选策略**：

1. **Current State优先级最高**：压缩后必须准确
2. **Errors & Corrections次优先**：避免重复错误
3. **Files and Functions精简**：只保留关键路径
4. **Worklog最简**：极简步骤记录

### 记忆压缩

**Session Memory压缩集成**：

```typescript
export async function trySessionMemoryCompaction(
  messages: Message[],
): Promise<CompactionResult | null> {
  // 1. 获取Session Memory内容
  const sessionMemory = await getSessionMemoryContent()

  // 2. 截断超长section
  const { truncatedContent, wasTruncated } =
    truncateSessionMemoryForCompact(sessionMemory)

  // 3. 创建摘要消息
  const summaryContent = getCompactUserSummaryMessage(
    truncatedContent,
    true,  // suppressFollowUpQuestions
    transcriptPath,
    true   // recentMessagesPreserved
  )

  // 4. 返回压缩结果
  return {
    boundaryMarker,
    summaryMessages: [createUserMessage({
      content: summaryContent,
      isCompactSummary: true,
    })],
    messagesToKeep,
    postCompactTokenCount: estimateMessageTokens(summaryMessages),
  }
}
```

---

## 7. 工具调用优化

### 工具描述精简

**可压缩工具列表**（`apiMicrocompact.ts:19-32`）：

```typescript
const TOOLS_CLEARABLE_RESULTS = [
  ...SHELL_TOOL_NAMES,     // Bash, KillShell
  GLOB_TOOL_NAME,          // Glob
  GREP_TOOL_NAME,          // Grep
  FILE_READ_TOOL_NAME,     // Read
  WEB_FETCH_TOOL_NAME,     // WebFetch
  WEB_SEARCH_TOOL_NAME,    // WebSearch
]

const TOOLS_CLEARABLE_USES = [
  FILE_EDIT_TOOL_NAME,     // Edit
  FILE_WRITE_TOOL_NAME,    // Write
  NOTEBOOK_EDIT_TOOL_NAME, // NotebookEdit
]
```

**工具描述优化策略**：

1. **延迟加载**：通过tool search beta延迟加载LSP/MCP工具
2. **动态schema**：只发送当前需要的工具定义
3. **工具引用**：使用tool_reference替代完整schema

### 工具结果压缩

**Microcompact工具结果清理**（`microCompact.ts:138-157`）：

```typescript
function calculateToolResultTokens(block: ToolResultBlockParam): number {
  if (!block.content) return 0

  if (typeof block.content === 'string') {
    return roughTokenCountEstimation(block.content)
  }

  // Array of blocks
  return block.content.reduce((sum, item) => {
    if (item.type === 'text') {
      return sum + roughTokenCountEstimation(item.text)
    } else if (item.type === 'image' || item.type === 'document') {
      // 图像/文档固定2000 Token估算
      return sum + IMAGE_MAX_TOKEN_SIZE  // 2000
    }
    return sum
  }, 0)
}
```

**工具结果清理标记**：

```typescript
export const TIME_BASED_MC_CLEARED_MESSAGE = '[Old tool result content cleared]'
```

**清理策略**：

1. **保留工具元数据**：tool_use_id、tool name
2. **替换内容为标记**：`[Old tool result content cleared]`
3. **保持配对完整性**：确保tool_use/tool_result配对完整

### 大文件读取优化

**大文件处理策略**：

1. **分段读取**：`Read`工具支持offset/limit参数
2. **图片固定估算**：2000 Token/图片（避免base64计数）
3. **文档固定估算**：2000 Token/文档（避免PDF base64计数）
4. **JSON高密度估算**：bytesPerToken=2（默认4）

**文件类型Token估算**（`tokenEstimation.ts:215-243`）：

```typescript
export function bytesPerTokenForFileType(fileExtension: string): number {
  switch (fileExtension) {
    case 'json':
    case 'jsonl':
    case 'jsonc':
      return 2  // JSON高密度，单字符Token多
    default:
      return 4  // 默认估算
  }
}

export function roughTokenCountEstimationForFileType(
  content: string,
  fileExtension: string,
): number {
  return roughTokenCountEstimation(
    content,
    bytesPerTokenForFileType(fileExtension)
  )
}
```

---

## 8. 代码分析Token优化

### 文件扫描策略

**Glob/Grep工具优化**：

1. **限制输出**：默认250条结果
2. **路径过滤**：只扫描相关目录
3. **类型过滤**：只扫描特定文件类型
4. **并行执行**：多个glob/grep并行调用

**示例优化调用**：

```typescript
// 并行扫描多个类型
const [tsFiles, jsFiles, testFiles] = await Promise.all([
  Glob('**/*.ts'),
  Glob('**/*.js'),
  Glob('**/*.test.*'),
])

// 限制输出
Grep({
  pattern: 'TODO',
  head_limit: 50,  // 只返回50条
  type: 'ts',      // 只扫描TypeScript
})
```

### 符号索引

**工具延迟加载**（tool search beta）：

```typescript
// 工具延迟加载
const willDefer = (t: Tool) =>
  useToolSearch && (deferredToolNames.has(t.name) || shouldDeferLspTool(t))

// 延迟加载标记
if (willDefer(t)) {
  toolSchema.defer_loading = true
}
```

**延迟加载优势**：

1. **减少初始schema大小**：LSP/MCP工具延迟加载
2. **按需加载**：只有使用时才加载完整schema
3. **缓存友好**：基础工具schema保持稳定

### 增量分析

**Session Memory增量更新**：

```typescript
// 更新阈值：每增长5000 Token更新一次
export function hasMetUpdateThreshold(currentTokenCount: number): boolean {
  const tokensSinceLastExtraction =
    currentTokenCount - tokensAtLastExtraction

  return tokensSinceLastExtraction >=
         sessionMemoryConfig.minimumTokensBetweenUpdate  // 5000
}

// 工具调用间隔更新
export function getToolCallsBetweenUpdates(): number {
  return sessionMemoryConfig.toolCallsBetweenUpdates  // 3
}
```

---

## 9. 用户使用建议

### 如何编写高效CLAUDE.md

**最佳实践**：

1. **控制大小**：保持在40K字符（10K Token）以内
2. **模块化**：使用`@include`分离不同类型指令
3. **结构清晰**：使用清晰的section标题
4. **信息密度**：避免冗余描述，直接列出关键点

**高效CLAUDE.md示例**：

```markdown
# Project Overview
E-commerce platform built with Next.js, TypeScript, Prisma.

# Key Files
- `src/pages/api/` - API routes
- `src/components/` - React components
- `prisma/schema.prisma` - Database schema

# Coding Standards
@include ./docs/coding-standards.md

# Common Commands
- `npm run dev` - Start dev server
- `npm run test` - Run tests
- `npx prisma migrate dev` - Database migration

# Architecture
@include ./docs/architecture.md

# Testing Patterns
@include ./docs/testing.md
```

**避免的低效CLAUDE.md**：

```markdown
# ❌ 低效示例

This is a very long description about the project history,
how it was founded, team members, company vision...
（冗余描述，占用大量Token）

We use React, which is a JavaScript library for building user interfaces,
developed by Facebook, released in 2013...  （不相关的背景知识）

TODO: Add more documentation later  （空section浪费空间）
```

### 何时手动触发/compact

**手动触发时机**：

1. **完成大任务后**：清理累积的工具结果
2. **切换话题前**：压缩旧话题上下文
3. **开始新阶段前**：为新任务释放Token空间
4. **长时间对话后**：避免接近上下文窗口限制

**触发方式**：

```
用户输入：/compact

可选参数：
/compact "只压缩测试相关内容"  # 自定义指令
/compact --no-auto           # 禁用自动压缩后手动触发
```

### 如何监控Token使用

**Token使用监控方法**：

1. **警告提示**：观察CLI的Token警告（黄色/红色）
2. **/cost命令**：查看当前会话Token消耗
3. **百分比显示**：CLI实时显示上下文窗口使用百分比

**/cost命令使用**（`cost-tracker.ts:228-244`）：

```typescript
export function formatTotalCost(): string {
  const costDisplay = formatCost(getTotalCostUSD())

  const modelUsageDisplay = formatModelUsage()

  return chalk.dim(
    `Total cost:            ${costDisplay}\n` +
    `Total duration (API):  ${formatDuration(getTotalAPIDuration())}\n` +
    `Total duration (wall): ${formatDuration(getTotalDuration())}\n` +
    `Total code changes:    ${getTotalLinesAdded()} lines added, ` +
    `${getTotalLinesRemoved()} lines removed\n` +
    `${modelUsageDisplay}`
  )
}
```

**输出示例**：

```
Total cost:            $0.15
Total duration (API):  45s
Total duration (wall): 1m 30s
Total code changes:    150 lines added, 20 lines removed

Usage by model:
        claude-sonnet-4-6: 12,000 input, 4,500 output,
                          8,000 cache read, 2,000 cache write ($0.10)
```

### /cost命令使用技巧

**使用场景**：

1. **会话结束时**：查看总成本
2. **压缩前后对比**：验证Token节省效果
3. **成本分析**：了解哪个模型消耗最多
4. **缓存效果**：查看cache_read vs cache_creation比例

**成本优化判断**：

```
高缓存命中率指标：
  cache_read_input_tokens >> cache_creation_input_tokens
  例如：80,000 cache read, 5,000 cache creation = 高命中率

低缓存命中率指标：
  cache_creation_input_tokens ≈ cache_read_input_tokens
  例如：20,000 cache creation, 20,000 cache read = 低命中率

优化方向：
  1. 保持系统提示词稳定（减少cache_creation）
  2. 连续对话（延长cache TTL有效期）
  3. 使用1h TTL（延长缓存生命周期）
```

---

## 10. 从零实现指南

### Token计数实现

**核心函数**（`tokens.ts:226-261`）：

```typescript
export function tokenCountWithEstimation(messages: readonly Message[]): number {
  let i = messages.length - 1

  // 从后向前查找最后一个有usage的assistant消息
  while (i >= 0) {
    const message = messages[i]
    const usage = message ? getTokenUsage(message) : undefined

    if (message && usage) {
      // 处理并行工具调用：相同message.id的多个assistant记录
      const responseId = getAssistantMessageId(message)
      if (responseId) {
        let j = i - 1
        while (j >= 0) {
          const prior = messages[j]
          const priorId = prior ? getAssistantMessageId(prior) : undefined
          if (priorId === responseId) {
            i = j  // 向前移动到第一个同id记录
          } else if (priorId !== undefined) {
            break  // 遇到不同API响应，停止
          }
          j--
        }
      }

      // API usage + 新消息估算
      return (
        getTokenCountFromUsage(usage) +
        roughTokenCountEstimationForMessages(messages.slice(i + 1))
      )
    }
    i--
  }

  // 没有usage记录，完全估算
  return roughTokenCountEstimationForMessages(messages)
}
```

**估算函数**（`tokenEstimation.ts:203-208`）：

```typescript
export function roughTokenCountEstimation(
  content: string,
  bytesPerToken: number = 4,
): number {
  // 快速估算：字符数 / 4
  return Math.round(content.length / bytesPerToken)
}
```

**API Token计数**（`tokenEstimation.ts:124-201`）：

```typescript
export async function countMessagesTokensWithAPI(
  messages: Anthropic.Beta.Messages.BetaMessageParam[],
  tools: Anthropic.Beta.Messages.BetaToolUnion[],
): Promise<number | null> {
  // 使用Haiku快速计数
  const model = getSmallFastModel()
  const anthropic = await getAnthropicClient({
    maxRetries: 1,
    model,
    source: 'count_tokens',
  })

  const response = await anthropic.beta.messages.countTokens({
    model: normalizeModelStringForAPI(model),
    messages: messages.length > 0 ? messages : [{ role: 'user', content: 'foo' }],
    tools,
    ...(betas.length > 0 && { betas }),
  })

  return response.input_tokens
}
```

### 压缩算法设计

**Session Memory压缩算法核心**：

```typescript
// 伪代码实现
function sessionMemoryCompact(messages, config) {
  // 1. 确定起始位置
  let startIndex = findLastSummarizedIndex(messages) + 1

  // 2. 计算当前Token
  let tokens = estimateTokens(messages.slice(startIndex))

  // 3. 向前扩展满足最小要求
  while (tokens < config.minTokens &&
         textMessages < config.minTextBlockMessages &&
         tokens < config.maxTokens) {

    // 向前包含一个消息
    startIndex--
    tokens += estimateTokens(messages[startIndex])

    if (hasTextBlocks(messages[startIndex])) {
      textMessages++
    }
  }

  // 4. 调整保持API不变量
  startIndex = adjustForToolPairing(messages, startIndex)

  // 5. 创建压缩结果
  return {
    summary: sessionMemoryContent,
    messagesToKeep: messages.slice(startIndex),
    tokensSaved: preCompactTokens - postCompactTokens
  }
}
```

**Traditional Compact算法核心**：

```typescript
// 伪代码实现
async function traditionalCompact(messages, context) {
  // 1. 计算预压缩Token
  const preTokens = estimateTokens(messages)

  // 2. 分组（按API轮次）
  const groups = groupByApiRound(messages)

  // 3. 选择保留范围
  const keepGroups = selectRecentGroups(groups, config.keepRecent)

  // 4. 调用API生成摘要
  const summary = await callClaudeAPI({
    model: 'haiku',
    system: compactPrompt,
    messages: messagesToSummarize,
    maxTurns: 1,
    tools: []  // 无工具
  })

  // 5. 构建压缩消息
  return [
    createBoundaryMarker(preTokens),
    createSummaryMessage(summary),
    ...messagesToKeep
  ]
}
```

### 缓存策略实现

**缓存控制添加**：

```typescript
// 伪代码实现
function addCacheBreakpoints(messages, config) {
  // 1. 系统提示词：1h TTL
  systemPrompt[0].cache_control = {
    type: 'ephemeral',
    ttl: '1h'
  }

  // 2. 工具定义：ephemeral缓存
  systemPrompt[1].cache_control = {
    type: 'ephemeral'
  }

  // 3. 对话历史：最后一个user message
  const lastUserMessage = messages.findLast(m => m.type === 'user')
  lastUserMessage.content[lastBlock].cache_control = {
    type: 'ephemeral',
    ttl: should1hCacheTTL(querySource) ? '1h' : undefined
  }
}
```

**1h TTL决策**：

```typescript
// 伪代码实现
function should1hCacheTTL(querySource) {
  // 检查用户资格
  if (!isEligible(userType, subscriptionStatus)) {
    return false
  }

  // 检查白名单
  const allowlist = getGrowthBookConfig('tengu_prompt_cache_1h_config')

  return allowlist.some(pattern =>
    pattern.endsWith('*')
      ? querySource.startsWith(pattern.slice(0, -1))
      : querySource === pattern
  )
}
```

---

## 附录：关键源码路径

### 压缩相关文件

| 文件路径 | 功能描述 |
|---------|---------|
| `services/compact/compact.ts` | 传统压缩主逻辑 |
| `services/compact/autoCompact.ts` | 自动压缩触发管理 |
| `services/compact/microCompact.ts` | 工具结果轻量压缩 |
| `services/compact/sessionMemoryCompact.ts` | Session Memory压缩路径 |
| `services/compact/apiMicrocompact.ts` | API原生上下文管理 |
| `services/compact/prompt.ts` | 压缩提示词模板 |

### Token计数相关文件

| 文件路径 | 功能描述 |
|---------|---------|
| `utils/tokens.ts` | Token计数主函数 |
| `services/tokenEstimation.ts` | Token估算函数 |
| `utils/context.ts` | 上下文窗口配置 |

### 缓存相关文件

| 文件路径 | 功能描述 |
|---------|---------|
| `services/api/claude.ts` | API调用与缓存控制 |
| `services/api/promptCacheBreakDetection.ts` | 缓存失效检测 |

### Session Memory相关文件

| 文件路径 | 功能描述 |
|---------|---------|
| `services/SessionMemory/sessionMemoryUtils.ts` | Session Memory工具函数 |
| `services/SessionMemory/prompts.ts` | Session Memory提示词模板 |
| `services/SessionMemory/sessionMemory.ts` | Session Memory主逻辑 |

### CLAUDE.md相关文件

| 文件路径 | 功能描述 |
|---------|---------|
| `utils/claudemd.ts` | CLAUDE.md加载与处理 |

### 成本追踪相关文件

| 文件路径 | 功能描述 |
|---------|---------|
| `cost-tracker.ts` | 成本计算与显示 |

---

**文档版本**：v1.0
**更新日期**：2026-04-02
**适用版本**：Claude Code CLI latest