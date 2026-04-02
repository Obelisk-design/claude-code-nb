# Compact 消息压缩详解

**分析日期:** 2026-04-02

## 1. 概述

### 1.1 Compact 命令的作用

`/compact` 命令是 Claude Code CLI 中用于管理对话上下文长度的核心机制。当对话历史超过模型的上下文窗口限制时，compact 命令通过生成摘要来压缩历史消息，释放 token 空间以便继续对话。

**核心功能：**
- 将长对话历史压缩为结构化摘要
- 保留关键技术细节（文件路径、代码片段、错误修复）
- 维护对话连贯性，使模型能无缝继续工作
- 支持手动触发和自动触发两种模式

### 1.2 消息压缩的必要性（Token 限制）

**Token 限制背景：**
- Claude 模型有固定的上下文窗口（约 200K tokens）
- 每次对话都会消耗 token，长对话会耗尽上下文空间
- 超过限制会导致 `prompt_too_long` 错误，对话无法继续

**压缩解决的问题：**
```
services/compact/autoCompact.ts 第 72-91 行

export function getAutoCompactThreshold(model: string): number {
  const effectiveContextWindow = getEffectiveContextWindowSize(model)
  const autocompactThreshold = effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS
  // AUTOCOMPACT_BUFFER_TOKENS = 13_000 tokens
  // 有效上下文窗口 = 总窗口 - 输出预留（20_000）
  // 自动压缩阈值 = 有效窗口 - 缓冲（13_000）
  return autocompactThreshold
}
```

当 token 使用量超过阈值时，系统自动触发压缩，避免达到硬性限制。

---

## 2. commands/compact/ 目录逐行分析

### 2.1 commands/compact/index.ts 入口

**文件路径:** `commands/compact/index.ts`

```typescript
// 第 1-15 行：命令定义
const compact = {
  type: 'local',                    // 本地命令，不需要远程处理
  name: 'compact',                  // 命令名称
  description: 'Clear conversation history but keep a summary in context...',
  isEnabled: () => !isEnvTruthy(process.env.DISABLE_COMPACT),  // 可通过环境变量禁用
  supportsNonInteractive: true,     // 支持非交互模式
  argumentHint: '<optional custom summarization instructions>',  // 支持自定义指令
  load: () => import('./compact.js'),  // 动态加载实现
} satisfies Command
```

**关键设计点：**
- 使用 `isEnabled` 函数动态检查环境变量，允许在测试或特殊场景禁用
- `argumentHint` 提示用户可以传入自定义压缩指令，如 `/compact focus on code changes`
- 动态加载避免启动时加载所有命令实现

### 2.2 commands/compact/compact.ts 核心实现

**文件路径:** `commands/compact/compact.ts`

**主要函数: `call` (第 40-137 行)**

```typescript
export const call: LocalCommandCall = async (args, context) => {
  const { abortController } = context
  let { messages } = context

  // 第 46 行：过滤已压缩的消息
  messages = getMessagesAfterCompactBoundary(messages)

  if (messages.length === 0) {
    throw new Error('No messages to compact')
  }

  const customInstructions = args.trim()  // 用户自定义指令

  // 第 57-83 行：尝试 Session Memory 压缩（优先路径）
  if (!customInstructions) {
    const sessionMemoryResult = await trySessionMemoryCompaction(
      messages,
      context.agentId,
    )
    if (sessionMemoryResult) {
      // Session Memory 压缩成功，直接返回
      getUserContext.cache.clear?.()
      runPostCompactCleanup()
      suppressCompactWarning()
      return {
        type: 'compact',
        compactionResult: sessionMemoryResult,
        displayText: buildDisplayText(context),
      }
    }
  }

  // 第 97-99 行：执行 Microcompact（轻量压缩）
  const microcompactResult = await microcompactMessages(messages, context)
  const messagesForCompact = microcompactResult.messages

  // 第 101-108 行：传统压缩路径
  const result = await compactConversation(
    messagesForCompact,
    context,
    await getCacheSharingParams(context, messagesForCompact),
    false,        // 不抑制后续问题
    customInstructions,
    false,        // 不是自动压缩
  )

  // 后处理
  setLastSummarizedMessageId(undefined)
  suppressCompactWarning()
  getUserContext.cache.clear?.()
  runPostCompactCleanup()

  return {
    type: 'compact',
    compactionResult: result,
    displayText: buildDisplayText(context, result.userDisplayMessage),
  }
}
```

**压缩路径优先级：**
1. **Session Memory Compaction** - 如果可用且无自定义指令，优先使用（最快，无需 API 调用）
2. **Microcompact** - 清理工具结果中的冗余内容
3. **Traditional Compact** - 调用 API 生成摘要（最完整但最慢）

---

## 3. 压缩算法分析

### 3.1 消息选择策略

**文件路径:** `services/compact/grouping.ts`

```typescript
// 第 22-63 行：按 API 轮次分组
export function groupMessagesByApiRound(messages: Message[]): Message[][] {
  const groups: Message[][] = []
  let current: Message[] = []
  let lastAssistantId: string | undefined

  for (const msg of messages) {
    // 边界条件：新 assistant 消息开始 = 新 API 轮次
    if (
      msg.type === 'assistant' &&
      msg.message.id !== lastAssistantId &&
      current.length > 0
    ) {
      groups.push(current)
      current = [msg]
    } else {
      current.push(msg)
    }
    if (msg.type === 'assistant') {
      lastAssistantId = msg.message.id
    }
  }

  if (current.length > 0) {
    groups.push(current)
  }
  return groups
}
```

**分组原理：**
- 每个 API 调用产生一个 `assistant` 消息，具有唯一 `message.id`
- 按此 ID 分组确保 `tool_use/tool_result` 配对完整
- 避免在压缩时破坏 API 语义约束

### 3.2 保留关键信息

**文件路径:** `services/compact/prompt.ts`

**BASE_COMPACT_PROMPT (第 61-143 行) 定义了摘要必须包含的内容：**

```typescript
const BASE_COMPACT_PROMPT = `Your task is to create a detailed summary...

Your summary should include the following sections:

1. Primary Request and Intent: 用户的所有显式请求
2. Key Technical Concepts: 重要技术概念、框架
3. Files and Code Sections: 文件路径、代码片段、修改原因
4. Errors and fixes: 遇到的错误及修复方式
5. Problem Solving: 已解决的问题和进行中的调试
6. All user messages: 所有用户消息（非工具结果）
7. Pending Tasks: 待处理任务
8. Current Work: 最近正在处理的具体工作
9. Optional Next Step: 下一步计划（带原文引用）

<example>
<analysis>
[思考过程，确保覆盖所有要点]
</analysis>

<summary>
1. Primary Request and Intent:
   [详细描述]
...
</summary>
</example>
`
```

**关键保留信息：**
- 文件路径（完整路径，不是相对路径）
- 代码片段（重要函数签名、关键修改）
- 用户反馈（用户明确说"这样做不对"的指令）
- 错误和修复（避免重复错误）

### 3.3 Summary 生成

**文件路径:** `services/compact/compact.ts`

**streamCompactSummary 函数 (第 1136-1199 行):**

```typescript
async function streamCompactSummary({
  messages,
  summaryRequest,
  appState,
  context,
  preCompactTokenCount,
  cacheSafeParams,
}): Promise<AssistantMessage> {
  // 使用 forked agent 复用主对话的缓存前缀
  const promptCacheSharingEnabled = getFeatureValue_CACHED_MAY_BE_STALE(
    'tengu_compact_cache_prefix',
    true,
  )

  // 发送 keep-alive 信号防止 WebSocket 超时
  const activityInterval = setInterval(
    () => {
      sendSessionActivitySignal()
      context.setSDKStatus?.('compacting')
    },
    30_000,  // 每 30 秒
  )

  if (promptCacheSharingEnabled) {
    // Forked agent 路径：复用缓存
    const result = await runForkedAgent({
      promptMessages: [summaryRequest],
      cacheSafeParams,
      canUseTool: createCompactCanUseTool(),  // 禁止工具调用
      querySource: 'compact',
      maxTurns: 1,
      skipCacheWrite: true,
      overrides: { abortController: context.abortController },
    })
    return result
  }
  // 否则使用普通 streaming 路径...
}
```

**生成流程：**
1. 构造摘要请求消息（包含 BASE_COMPACT_PROMPT）
2. 使用 forked agent 复用主对话缓存（节省 API 成本）
3. 禁止工具调用（模型只能生成文本摘要）
4. 处理 `prompt_too_long` 错误时截断头部重试

---

## 4. utils/compact/ 相关实现

**注意：utils/compact/ 目录不存在，compact 相关工具函数分布在其他位置。**

### 4.1 消息处理工具

**文件路径:** `utils/messages.ts`

**getMessagesAfterCompactBoundary (第 4643-4648 行):**

```typescript
export function getMessagesAfterCompactBoundary<
  T extends Message | NormalizedMessage,
>(messages: T[], options?: { includeSnipped?: boolean }): T[] {
  const boundaryIndex = findLastCompactBoundaryIndex(messages)
  const sliced = boundaryIndex === -1 ? messages : messages.slice(boundaryIndex)
  // 过滤已压缩的消息，只保留边界后的内容
  return sliced
}
```

**createCompactBoundaryMessage (第 4530-4560 行):**

```typescript
export function createCompactBoundaryMessage(
  trigger: 'manual' | 'auto',
  preTokens: number,
  lastPreCompactMessageUuid?: UUID,
  userContext?: string,
  messagesSummarized?: number,
): SystemCompactBoundaryMessage {
  return {
    type: 'system',
    subtype: 'compact_boundary',
    uuid: randomUUID(),
    timestamp: new Date().toISOString(),
    compactMetadata: {
      trigger,
      preCompactTokenCount: preTokens,
      lastPreCompactMessageUuid,
      userContext,
      messagesSummarized,
    },
  }
}
```

**边界消息的作用：**
- 标记压缩发生的位置
- 存储压缩前的 token 数量（用于统计）
- 记录触发类型（手动/自动）

### 4.2 Token 计算

**文件路径:** `services/compact/microCompact.ts`

**estimateMessageTokens (第 164-205 行):**

```typescript
export function estimateMessageTokens(messages: Message[]): number {
  let totalTokens = 0

  for (const message of messages) {
    if (message.type !== 'user' && message.type !== 'assistant') {
      continue
    }

    for (const block of message.message.content) {
      if (block.type === 'text') {
        totalTokens += roughTokenCountEstimation(block.text)
      } else if (block.type === 'tool_result') {
        totalTokens += calculateToolResultTokens(block)
      } else if (block.type === 'image' || block.type === 'document') {
        totalTokens += IMAGE_MAX_TOKEN_SIZE  // 2000
      } else if (block.type === 'thinking') {
        totalTokens += roughTokenCountEstimation(block.thinking)
      }
      // ... 其他类型
    }
  }

  // 保守估计：乘以 4/3
  return Math.ceil(totalTokens * (4 / 3))
}
```

---

## 5. QueryEngine 中的压缩触发

### 5.1 自动压缩触发条件

**文件路径:** `services/compact/autoCompact.ts`

**shouldAutoCompact 函数 (第 160-239 行):**

```typescript
export async function shouldAutoCompact(
  messages: Message[],
  model: string,
  querySource?: QuerySource,
  snipTokensFreed = 0,
): Promise<boolean> {
  // 递归保护：session_memory 和 compact 是 forked agent，会死锁
  if (querySource === 'session_memory' || querySource === 'compact') {
    return false
  }

  // 禁用检查
  if (!isAutoCompactEnabled()) {
    return false
  }

  // 计算 token 数量
  const tokenCount = tokenCountWithEstimation(messages) - snipTokensFreed
  const threshold = getAutoCompactThreshold(model)

  const { isAboveAutoCompactThreshold } = calculateTokenWarningState(
    tokenCount,
    model,
  )

  return isAboveAutoCompactThreshold
}
```

**触发阈值计算：**

```
有效上下文窗口 = 模型窗口 - 输出预留（20,000）
自动压缩阈值 = 有效窗口 - 缓冲（13,000）

示例（200K 窗口）：
- 有效窗口 = 200,000 - 20,000 = 180,000
- 阈值 = 180,000 - 13,000 = 167,000
- 当 token > 167,000 时触发自动压缩
```

### 5.2 手动压缩入口

**文件路径:** `commands/compact/compact.ts` 的 `call` 函数

手动压缩通过 `/compact` 命令触发，流程：

```typescript
// 1. 过滤已压缩消息
messages = getMessagesAfterCompactBoundary(messages)

// 2. 尝试 Session Memory 压缩（快路径）
const sessionMemoryResult = await trySessionMemoryCompaction(messages, context.agentId)

// 3. 执行 Microcompact
const microcompactResult = await microcompactMessages(messages, context)

// 4. 调用 API 生成摘要
const result = await compactConversation(messagesForCompact, context, cacheSafeParams, false, customInstructions, false)

// 5. 后处理清理
runPostCompactCleanup()
```

---

## 6. 压缩前后对比

### 6.1 Token 节省效果

**典型压缩效果（来自 compact.ts 第 650-695 行的日志）：**

```typescript
logEvent('tengu_compact', {
  preCompactTokenCount,           // 压缩前：如 150,000 tokens
  postCompactTokenCount,          // 压缩后 API 总使用：如 152,000
  truePostCompactTokenCount,      // 实际结果上下文：如 8,000 tokens
  compactionInputTokens,          // 压缩 API 输入
  compactionOutputTokens,         // 摘要输出：如 3,000 tokens
  compactionCacheReadTokens,      // 缓存读取节省
})
```

**典型节省：**
- 压缩前：150,000 tokens
- 摘要生成：约 3,000 tokens
- 附件恢复：约 5,000 tokens（文件内容、技能等）
- 压缩后总计：约 8,000 tokens
- 节省：约 142,000 tokens（95% 减少）

### 6.2 信息损失权衡

**保留的信息：**
- 所有用户请求和意图
- 关键文件路径和代码片段
- 错误及修复方法
- 待处理任务
- 最近工作内容

**丢失的信息：**
- 工具调用详细输出（被替换为 `[Old tool result content cleared]`）
- 中间思考过程
- 完整对话原文（可从 transcript 文件恢复）

**恢复机制：**
```typescript
// prompt.ts 第 349-351 行
baseSummary += `\n\nIf you need specific details from before compaction,
read the full transcript at: ${transcriptPath}`
```

用户可通过读取 transcript 文件获取完整历史。

---

## 7. 从零实现指南

### 7.1 如何设计消息压缩

**核心架构：**

```
┌─────────────────────────────────────────────────────────────┐
│                     Compact 系统架构                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │ 触发检测    │───>│ 消息分组    │───>│ 摘要生成    │     │
│  │ (autoCompact)│    │ (grouping)  │    │ (prompt.ts) │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│        │                  │                  │              │
│        v                  v                  v              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │ 阈值计算    │    │ 边界标记    │    │ 后处理清理  │     │
│  │ (threshold) │    │ (boundary)  │    │ (cleanup)   │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 选择策略

**实现选择策略的核心代码：**

```typescript
// 伪代码：消息选择策略
function selectMessagesToCompact(messages: Message[], threshold: number): {
  toSummarize: Message[]
  toKeep: Message[]
} {
  // 1. 按 API 轮次分组
  const groups = groupMessagesByApiRound(messages)

  // 2. 从头部开始累积 token
  let accumulatedTokens = 0
  let summarizeEndIndex = 0

  for (const group of groups) {
    const groupTokens = estimateMessageTokens(group)
    accumulatedTokens += groupTokens

    // 停止条件：保留足够上下文
    if (accumulatedTokens >= threshold * 0.9) {
      break
    }
    summarizeEndIndex++
  }

  // 3. 分割
  const toSummarize = groups.slice(0, summarizeEndIndex).flat()
  const toKeep = groups.slice(summarizeEndIndex).flat()

  return { toSummarize, toKeep }
}
```

**关键约束：**
- 不分割 `tool_use/tool_result` 配对
- 保留最近的 5 个带文本的消息
- 最少保留 10,000 tokens
- 最多保留 40,000 tokens（sessionMemoryCompact 配置）

### 7.3 Summary 生成模板

**最小实现模板：**

```typescript
const COMPACT_PROMPT_TEMPLATE = `
你需要创建对话的详细摘要。

摘要必须包含：
1. 用户请求：所有显式请求和意图
2. 技术概念：使用的技术、框架、库
3. 文件和代码：涉及的文件路径和关键代码片段
4. 错误修复：遇到的错误及解决方案
5. 当前任务：最近正在处理的工作
6. 下一步：计划执行的操作

<example>
<analysis>
[你的思考过程]
</analysis>

<summary>
1. 用户请求：[描述]
2. 技术概念：[列表]
3. 文件和代码：
   - path/to/file.ts
     [关键代码]
4. 错误修复：[描述]
5. 当前任务：[描述]
6. 下一步：[描述]
</summary>
</example>

请生成摘要。
`

async function generateSummary(messages: Message[]): Promise<string> {
  const response = await callAPI({
    messages: [
      ...messages,
      { role: 'user', content: COMPACT_PROMPT_TEMPLATE }
    ],
    maxTokens: 4000,
  })
  return extractSummary(response)
}

function extractSummary(response: string): string {
  // 提取 <summary> 标签内容
  const match = response.match(/<summary>([\s\S]*?)<\/summary>/)
  return match ? match[1].trim() : response
}
```

---

## 附录：关键文件路径索引

| 功能 | 文件路径 |
|------|----------|
| 命令入口 | `commands/compact/index.ts` |
| 命令实现 | `commands/compact/compact.ts` |
| 自动压缩 | `services/compact/autoCompact.ts` |
| 核心压缩逻辑 | `services/compact/compact.ts` |
| 摘要提示词 | `services/compact/prompt.ts` |
| 消息分组 | `services/compact/grouping.ts` |
| Session Memory 压缩 | `services/compact/sessionMemoryCompact.ts` |
| Microcompact | `services/compact/microCompact.ts` |
| 后清理 | `services/compact/postCompactCleanup.ts` |
| 消息工具 | `utils/messages.ts` |
| Token 计算 | `services/compact/microCompact.ts` (estimateMessageTokens) |
| QueryEngine 集成 | `query.ts` |

---

*Compact 消息压缩详解 - 2026-04-02*