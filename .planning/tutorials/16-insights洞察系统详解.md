# Insights 洞察系统详解

**分析日期:** 2026-04-02

## 1. 概述

### 1.1 Insights 系统的设计目标

Claude Code 的 Insights 系统是一个**用户行为分析与洞察生成平台**，其核心设计目标包括：

1. **会话数据收集**：从用户的 Claude Code 使用记录中提取结构化数据
2. **智能分析**：使用 AI 模型（Opus）分析用户交互模式、成功因素和摩擦点
3. **个性化洞察**：生成针对特定用户的使用报告，包括改进建议和新功能推荐
4. **可视化呈现**：生成可分享的 HTML 报告，包含图表和交互式元素

### 1.2 用户行为分析维度

系统从以下维度分析用户行为：

| 维度 | 描述 | 数据来源 |
|------|------|----------|
| **工具使用统计** | 各工具调用次数、错误率 | `extractToolStats()` |
| **语言分布** | 编辑的文件语言类型统计 | 文件扩展名映射 |
| **Git 活动** | commit/push 次数 | Bash 工具命令解析 |
| **会话元数据** | 持续时间、消息数量、Token 使用量 | 日志文件解析 |
| **用户响应时间** | 用户回复 Claude 的平均时间 | 时间戳差值计算 |
| **摩擦分析** | 工具失败、用户中断、误解请求 | 错误分类算法 |
| **多 Claude 检测** | 同时使用多个会话的模式 | 滑动窗口算法 |

---

## 2. commands/insights.ts 命令分析

### 2.1 命令入口定义

**文件路径**: `commands/insights.ts`

```typescript
// 命令默认导出
export default usageReport

// 命令定义结构
const usageReport: Command = {
  name: 'insights',
  description: 'Generate insights from Claude Code usage',
  hidden: true,  // 内部命令，不对外暴露
  action: async () => {
    // 1. 收集会话数据
    // 2. 提取 facets（结构化特征）
    // 3. 调用 AI 生成洞察
    // 4. 生成 HTML 报告
    // 5. 上传并返回 URL
  }
}
```

### 2.2 数据收集流程

```typescript
// 会话元数据类型定义
type SessionMeta = {
  session_id: string
  project_path: string
  start_time: string
  duration_minutes: number
  user_message_count: number
  assistant_message_count: number
  tool_counts: Record<string, number>
  languages: Record<string, number>
  git_commits: number
  git_pushes: number
  input_tokens: number
  output_tokens: number
  // 新增统计维度
  user_interruptions: number
  user_response_times: number[]
  tool_errors: number
  tool_error_categories: Record<string, number>
  uses_task_agent: boolean
  uses_mcp: boolean
  uses_web_search: boolean
  uses_web_fetch: boolean
  lines_added: number
  lines_removed: number
  files_modified: number
  message_hours: number[]
  user_message_timestamps: string[]
}
```

### 2.3 工具统计提取核心逻辑

**函数**: `extractToolStats()` (行 467-728)

```typescript
function extractToolStats(log: LogOption): {
  toolCounts: Record<string, number>
  languages: Record<string, number>
  // ...其他统计
} {
  const toolCounts: Record<string, number> = {}
  const languages: Record<string, number> = {}
  let gitCommits = 0, gitPushes = 0
  let inputTokens = 0, outputTokens = 0
  
  // 遍历消息提取统计
  for (const msg of log.messages) {
    if (msg.type === 'assistant' && msg.message) {
      const content = msg.message.content
      if (Array.isArray(content)) {
        for (const block of content) {
          if (block.type === 'tool_use' && 'name' in block) {
            const toolName = block.name as string
            toolCounts[toolName] = (toolCounts[toolName] || 0) + 1
            
            // 检测特殊工具使用
            if (toolName === AGENT_TOOL_NAME) usesTaskAgent = true
            if (toolName.startsWith('mcp__')) usesMcp = true
            if (toolName === 'WebSearch') usesWebSearch = true
            
            // 提取语言统计（从文件路径）
            const filePath = input.file_path
            if (filePath) {
              const lang = getLanguageFromPath(filePath)
              if (lang) languages[lang] = (languages[lang] || 0) + 1
            }
            
            // 计算代码行数变化（Edit/Write 工具）
            if (toolName === 'Edit') {
              const changes = diffLines(oldString, newString)
              // 累加 added/removed 行数
            }
          }
        }
      }
    }
  }
}
```

### 2.4 Facet 提取（AI 分析）

**函数**: `extractFacetsFromAPI()` (行 1001-1055)

系统使用 Opus 模型从会话转录中提取结构化特征：

```typescript
async function extractFacetsFromAPI(
  log: LogOption,
  sessionId: string,
): Promise<SessionFacets | null> {
  // 长会话需要分块摘要
  const transcript = await formatTranscriptWithSummarization(log)
  
  // 构建分析 prompt
  const jsonPrompt = `${FACET_EXTRACTION_PROMPT}${transcript}

RESPOND WITH ONLY A VALID JSON OBJECT matching this schema:
{
  "underlying_goal": "What the user fundamentally wanted to achieve",
  "goal_categories": {"category_name": count, ...},
  "outcome": "fully_achieved|mostly_achieved|...",
  "user_satisfaction_counts": {"level": count, ...},
  "claude_helpfulness": "unhelpful|slightly_helpful|...",
  "session_type": "single_task|multi_task|...",
  "friction_counts": {"friction_type": count, ...},
  "friction_detail": "One sentence describing friction or empty",
  "primary_success": "none|fast_accurate_search|...",
  "brief_summary": "One sentence: what user wanted and whether they got it"
}`
  
  // 调用 API
  const result = await queryWithModel({
    model: getAnalysisModel(),  // Opus
    userPrompt: jsonPrompt,
    options: { querySource: 'insights', ... }
  })
  
  // 解析 JSON 响应
  const parsed = jsonParse(jsonMatch[0])
  return { ...parsed, session_id: sessionId }
}
```

### 2.5 多 Claude 检测算法

**函数**: `detectMultiClauding()` (行 1062-1143)

使用滑动窗口检测用户同时使用多个 Claude 会话的模式：

```typescript
export function detectMultiClauding(
  sessions: Array<{
    session_id: string
    user_message_timestamps: string[]
  }>,
): {
  overlap_events: number
  sessions_involved: number
  user_messages_during: number
} {
  const OVERLAP_WINDOW_MS = 30 * 60000  // 30分钟窗口
  
  // 收集所有会话的消息时间戳
  const allSessionMessages: Array<{ ts: number; sessionId: string }> = []
  for (const session of sessions) {
    for (const timestamp of session.user_message_timestamps) {
      allSessionMessages.push({
        ts: new Date(timestamp).getTime(),
        sessionId: session.session_id
      })
    }
  }
  
  // 按时间排序
  allSessionMessages.sort((a, b) => a.ts - b.ts)
  
  // 滑动窗口检测 pattern: s1 -> s2 -> s1
  for (let i = 0; i < allSessionMessages.length; i++) {
    // 收缩窗口左侧
    while (msg.ts - allSessionMessages[windowStart]!.ts > OVERLAP_WINDOW_MS) {
      windowStart++
    }
    
    // 检测是否在窗口内出现相同会话（表示交替使用）
    const prevIndex = sessionLastIndex.get(msg.sessionId)
    if (prevIndex !== undefined) {
      // 发现重叠事件
      multiClaudeSessionPairs.add(pair)
    }
  }
}
```

### 2.6 并行洞察生成

系统定义 6+ 个并行执行的洞察生成任务：

```typescript
const INSIGHT_SECTIONS: InsightSection[] = [
  {
    name: 'project_areas',
    prompt: `Analyze project areas...`,
    maxTokens: 8192,
  },
  {
    name: 'interaction_style',
    prompt: `Analyze interaction style...`,
    maxTokens: 8192,
  },
  {
    name: 'what_works',
    prompt: `Analyze what's working well...`,
    maxTokens: 8192,
  },
  {
    name: 'friction_analysis',
    prompt: `Analyze friction points...`,
    maxTokens: 8192,
  },
  {
    name: 'suggestions',
    prompt: `Suggest improvements...`,
    maxTokens: 8192,
  },
  {
    name: 'on_the_horizon',
    prompt: `Identify future opportunities...`,
    maxTokens: 8192,
  },
  // ant-only sections
  ...(process.env.USER_TYPE === 'ant' ? [
    { name: 'cc_team_improvements', ... },
    { name: 'model_behavior_improvements', ... },
  ] : []),
]
```

---

## 3. services/analytics/ 分析

### 3.1 index.ts - 公共 API 入口

**文件路径**: `services/analytics/index.ts`

这是分析服务的**主入口模块**，设计上**零依赖**以避免循环导入：

```typescript
/**
 * Marker type for verifying analytics metadata doesn't contain sensitive data
 * 使用此类型强制验证字符串值不包含代码片段或文件路径
 */
export type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS = never

/**
 * Marker type for PII-tagged values routed to privileged proto columns
 * 用于标记需要路由到特权 BQ 列的数据
 */
export type AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED = never

/**
 * Strip `_PROTO_*` keys from payload for general-access storage
 * 在发送到 Datadog 前剥离 PII 标记字段
 */
export function stripProtoFields<V>(
  metadata: Record<string, V>,
): Record<string, V> {
  let result: Record<string, V> | undefined
  for (const key in metadata) {
    if (key.startsWith('_PROTO_')) {
      if (result === undefined) {
        result = { ...metadata }
      }
      delete result[key]
    }
  }
  return result ?? metadata
}

// Sink 接口定义
export type AnalyticsSink = {
  logEvent: (eventName: string, metadata: LogEventMetadata) => void
  logEventAsync: (eventName: string, metadata: LogEventMetadata) => Promise<void>
}

// 事件队列（sink 未连接时暂存事件）
const eventQueue: QueuedEvent[] = []
let sink: AnalyticsSink | null = null

/**
 * Attach the analytics sink during app startup
 * 使用 queueMicrotask 异步排空队列，避免阻塞启动
 */
export function attachAnalyticsSink(newSink: AnalyticsSink): void {
  if (sink !== null) return  // 幂等操作
  sink = newSink
  
  if (eventQueue.length > 0) {
    const queuedEvents = [...eventQueue]
    eventQueue.length = 0
    
    queueMicrotask(() => {
      for (const event of queuedEvents) {
        if (event.async) {
          void sink!.logEventAsync(event.eventName, event.metadata)
        } else {
          sink!.logEvent(event.eventName, event.metadata)
        }
      }
    })
  }
}

/**
 * Log event (synchronous)
 * 自动采样、添加 sample_rate 元数据
 */
export function logEvent(eventName: string, metadata: LogEventMetadata): void {
  if (sink === null) {
    eventQueue.push({ eventName, metadata, async: false })
    return
  }
  sink.logEvent(eventName, metadata)
}
```

### 3.2 growthbook.ts - 特性开关

**文件路径**: `services/analytics/growthbook.ts`

GrowthBook 是**远程特性开关系统**，用于动态控制分析行为：

```typescript
import { GrowthBook } from '@growthbook/growthbook'

// 用户属性用于特性开关定向
export type GrowthBookUserAttributes = {
  id: string
  sessionId: string
  deviceID: string
  platform: 'win32' | 'darwin' | 'linux'
  apiBaseUrlHost?: string
  organizationUUID?: string
  accountUUID?: string
  userType?: string
  subscriptionType?: string
  rateLimitTier?: string
  firstTokenTime?: number
  email?: string
  appVersion?: string
  github?: GitHubActionsMetadata
}

// GrowthBook 客户端实例
let client: GrowthBook | null = null

// 刷新监听器信号
const refreshed = createSignal()

/**
 * Register callback for GrowthBook refresh events
 * 用于配置变更后重建依赖组件
 */
export function onGrowthBookRefresh(
  listener: GrowthBookRefreshListener,
): () => void {
  let subscribed = true
  const unsubscribe = refreshed.subscribe(() => callSafe(listener))
  
  // 如果已初始化，在下一个微任务触发监听器
  if (remoteEvalFeatureValues.size > 0) {
    queueMicrotask(() => {
      if (subscribed && remoteEvalFeatureValues.size > 0) {
        callSafe(listener)
      }
    })
  }
  
  return () => {
    subscribed = false
    unsubscribe()
  }
}

// 动态配置缓存
const remoteEvalFeatureValues = new Map<string, unknown>()

/**
 * Get feature value with cached fallback
 * 用于读取特性开关值，支持缓存回退
 */
export function getFeatureValue_CACHED_MAY_BE_STALE<T>(
  featureKey: string,
  defaultValue: T,
): T {
  // 优先使用环境变量覆盖（ant-only）
  const overrides = getEnvOverrides()
  if (overrides && overrides[featureKey] !== undefined) {
    return overrides[featureKey] as T
  }
  
  // 使用远程评估缓存
  if (remoteEvalFeatureValues.has(featureKey)) {
    return remoteEvalFeatureValues.get(featureKey) as T
  }
  
  return defaultValue
}
```

### 3.3 datadog.ts - 日志后端

**文件路径**: `services/analytics/datadog.ts`

Datadog 是**日志和分析后端**，用于事件追踪：

```typescript
const DATADOG_LOGS_ENDPOINT =
  'https://http-intake.logs.us5.datadoghq.com/api/v2/logs'
const DATADOG_CLIENT_TOKEN = 'pubbbf48e6d78dae54bceaa4acf463299bf'

// 批处理配置
const DEFAULT_FLUSH_INTERVAL_MS = 15000
const MAX_BATCH_SIZE = 100
const NETWORK_TIMEOUT_MS = 5000

// 允许的事件白名单
const DATADOG_ALLOWED_EVENTS = new Set([
  'chrome_bridge_connection_succeeded',
  'tengu_api_error',
  'tengu_api_success',
  'tengu_init',
  'tengu_tool_use_error',
  'tengu_tool_use_success',
  // ... 更多事件
])

// 标签字段（用于 Datadog 查询）
const TAG_FIELDS = [
  'arch', 'clientType', 'errorType', 'model', 'platform',
  'provider', 'userType', 'version', ...
]

// 日志批次
let logBatch: DatadogLog[] = []
let flushTimer: NodeJS.Timeout | null = null

/**
 * Track event to Datadog
 * 仅在生产环境且为 firstParty 提供者时发送
 */
export async function trackDatadogEvent(
  eventName: string,
  properties: { [key: string]: boolean | number | undefined },
): Promise<void> {
  if (process.env.NODE_ENV !== 'production') return
  if (getAPIProvider() !== 'firstParty') return  // 不发送 3P 提供者数据
  
  // 检查事件白名单
  if (!DATADOG_ALLOWED_EVENTS.has(eventName)) return
  
  // 获取事件元数据
  const metadata = await getEventMetadata({ model: properties.model })
  
  // 构建 Datadog 日志对象
  const log: DatadogLog = {
    ddsource: 'nodejs',
    ddtags: tags.join(','),
    message: eventName,
    service: 'claude-code',
    hostname: 'claude-code',
    env: process.env.USER_TYPE,
  }
  
  logBatch.push(log)
  
  // 批次满时立即刷新，否则调度刷新
  if (logBatch.length >= MAX_BATCH_SIZE) {
    void flushLogs()
  } else {
    scheduleFlush()
  }
}

/**
 * Flush logs to Datadog endpoint
 */
async function flushLogs(): Promise<void> {
  if (logBatch.length === 0) return
  const logsToSend = logBatch
  logBatch = []
  
  await axios.post(DATADOG_LOGS_ENDPOINT, logsToSend, {
    headers: {
      'Content-Type': 'application/json',
      'DD-API-KEY': DATADOG_CLIENT_TOKEN,
    },
    timeout: NETWORK_TIMEOUT_MS,
  })
}

/**
 * User bucket for cardinality reduction
 * 将用户 ID 映射到 30 个桶，用于估算用户数量而不暴露真实 ID
 */
const getUserBucket = memoize((): number => {
  const userId = getOrCreateUserID()
  const hash = createHash('sha256').update(userId).digest('hex')
  return parseInt(hash.slice(0, 8), 16) % NUM_USER_BUCKETS
})
```

### 3.4 firstPartyEventLoggingExporter.ts - 1P 事件导出器

**文件路径**: `services/analytics/firstPartyEventLoggingExporter.ts`

这是**第一方事件导出器**，实现 OpenTelemetry 的 `LogRecordExporter` 接口：

```typescript
import type { LogRecordExporter, ReadableLogRecord } from '@opentelemetry/sdk-logs'

/**
 * Exporter for 1st-party event logging to /api/event_logging/batch
 * 
 * 特性：
 * - 批处理导出（由 OTel BatchLogRecordProcessor 控制）
 * - 失败事件持久化到磁盘
 * - 二次方退避重试
 * - 认证回退（401 时重试无认证）
 */
export class FirstPartyEventLoggingExporter implements LogRecordExporter {
  private readonly endpoint: string
  private readonly maxBatchSize: number
  private readonly maxAttempts: number
  private pendingExports: Promise<void>[] = []
  private attempts = 0
  
  constructor(options: {
    timeout?: number
    maxBatchSize?: number
    skipAuth?: boolean
    maxAttempts?: number
    baseUrl?: string
    isKilled?: () => boolean  // killswitch 探测函数
  } = {}) {
    // 默认生产环境，staging 可覆盖
    const baseUrl = options.baseUrl ||
      (process.env.ANTHROPIC_BASE_URL === 'https://api-staging.anthropic.com'
        ? 'https://api-staging.anthropic.com'
        : 'https://api.anthropic.com')
    
    this.endpoint = `${baseUrl}/api/event_logging/batch`
    this.maxAttempts = options.maxAttempts ?? 8
    
    // 启动时重试上次失败的批次
    void this.retryPreviousBatches()
  }
  
  /**
   * Export logs (OTel interface)
   */
  async export(
    logs: ReadableLogRecord[],
    resultCallback: (result: ExportResult) => void,
  ): Promise<void> {
    // 过滤事件日志（按 scope name）
    const eventLogs = logs.filter(
      log => log.instrumentationScope?.name === 'com.anthropic.claude_code.events'
    )
    
    // 转换为事件格式
    const events = this.transformLogsToEvents(eventLogs).events
    
    // 批量发送
    const failedEvents = await this.sendEventsInBatches(events)
    
    if (failedEvents.length > 0) {
      await this.queueFailedEvents(failedEvents)
      this.scheduleBackoffRetry()
      resultCallback({ code: ExportResultCode.FAILED, ... })
    } else {
      resultCallback({ code: ExportResultCode.SUCCESS })
    }
  }
  
  /**
   * Send events with chunking and delay
   */
  private async sendEventsInBatches(
    events: FirstPartyEventLoggingEvent[],
  ): Promise<FirstPartyEventLoggingEvent[]> {
    const batches: FirstPartyEventLoggingEvent[][] = []
    for (let i = 0; i < events.length; i += this.maxBatchSize) {
      batches.push(events.slice(i, i + this.maxBatchSize))
    }
    
    // 首次失败时短路剩余批次
    for (let i = 0; i < batches.length; i++) {
      try {
        await this.sendBatchWithRetry({ events: batch })
      } catch (error) {
        // 短路：将剩余批次加入失败队列
        for (let j = i; j < batches.length; j++) {
          failedBatchEvents.push(...batches[j]!)
        }
        break
      }
    }
    
    return failedBatchEvents
  }
  
  /**
   * Quadratic backoff retry (matching Statsig SDK)
   */
  private scheduleBackoffRetry(): void {
    const delay = Math.min(
      this.baseBackoffDelayMs * this.attempts * this.attempts,
      this.maxBackoffDelayMs,
    )
    
    this.cancelBackoff = this.schedule(async () => {
      await this.retryFailedEvents()
    }, delay)
  }
  
  /**
   * Transform OTel logs to 1P event format
   */
  private transformLogsToEvents(
    logs: ReadableLogRecord[],
  ): FirstPartyEventLoggingPayload {
    const events: FirstPartyEventLoggingEvent[] = []
    
    for (const log of logs) {
      const attributes = log.attributes || {}
      const eventName = attributes.event_name as string
      
      // 提取核心元数据
      const coreMetadata = attributes.core_metadata as EventMetadata
      const userMetadata = attributes.user_metadata as CoreUserData
      
      // 处理 _PROTO_* PII 字段
      const { _PROTO_skill_name, _PROTO_plugin_name, ...rest } = formatted.additional
      const additionalMetadata = stripProtoFields(rest)
      
      events.push({
        event_type: 'ClaudeCodeInternalEvent',
        event_data: ClaudeCodeInternalEvent.toJSON({
          event_id: attributes.event_id,
          event_name: eventName,
          client_timestamp: this.hrTimeToDate(log.hrTime),
          device_id: attributes.user_id,
          email: userMetadata?.email,
          auth: formatted.auth,
          ...formatted.core,
          env: formatted.env,
          skill_name: _PROTO_skill_name,
          additional_metadata: Buffer.from(jsonStringify(additionalMetadata))
            .toString('base64'),
        }),
      })
    }
    
    return { events }
  }
}
```

### 3.5 metadata.ts - 元数据收集

**文件路径**: `services/analytics/metadata.ts`

这是**共享事件元数据收集模块**，为所有分析系统提供统一的元数据格式：

```typescript
/**
 * Core event metadata shared across all analytics systems
 */
export type EventMetadata = {
  model: string
  sessionId: string
  userType: string
  betas?: string
  envContext: EnvContext
  entrypoint?: string
  agentSdkVersion?: string
  isInteractive: string
  clientType: string
  processMetrics?: ProcessMetrics
  // Agent identification
  agentId?: string
  parentSessionId?: string
  agentType?: 'teammate' | 'subagent' | 'standalone'
  teamName?: string
  subscriptionType?: string
  rh?: string  // Repo remote hash
  kairosActive?: true
  skillMode?: 'discovery' | 'coach' | 'discovery_and_coach'
}

/**
 * Environment context metadata
 */
export type EnvContext = {
  platform: string
  platformRaw: string
  arch: string
  nodeVersion: string
  terminal: string | null
  packageManagers: string
  runtimes: string
  isRunningWithBun: boolean
  isCi: boolean
  isClaubbit: boolean
  isClaudeCodeRemote: boolean
  isLocalAgentMode: boolean
  isConductor: boolean
  remoteEnvironmentType?: string
  coworkerType?: string
  version: string
  versionBase?: string
  buildTime: string
  deploymentEnvironment: string
  // GitHub Actions metadata
  githubEventName?: string
  githubActionsRunnerOs?: string
  // Linux distro info
  linuxDistroId?: string
  linuxKernel?: string
  vcs?: string
}

/**
 * Get core event metadata (async)
 */
export async function getEventMetadata(
  options: EnrichMetadataOptions = {},
): Promise<EventMetadata> {
  const model = options.model ? String(options.model) : getMainLoopModel()
  const betas = getModelBetas(model).join(',')
  
  const [envContext, repoRemoteHash] = await Promise.all([
    buildEnvContext(),
    getRepoRemoteHash(),
  ])
  
  const metadata: EventMetadata = {
    model,
    sessionId: getSessionId(),
    userType: process.env.USER_TYPE || '',
    envContext,
    isInteractive: String(getIsInteractive()),
    clientType: getClientType(),
    sweBenchRunId: process.env.SWE_BENCH_RUN_ID || '',
    ...getAgentIdentification(),
    ...(getSubscriptionType() && { subscriptionType: getSubscriptionType()! }),
    ...(repoRemoteHash && { rh: repoRemoteHash }),
  }
  
  return metadata
}

/**
 * Build environment context (memoized)
 */
const buildEnvContext = memoize(async (): Promise<EnvContext> => {
  const [packageManagers, runtimes, linuxDistroInfo, vcs] = await Promise.all([
    env.getPackageManagers(),
    env.getRuntimes(),
    getLinuxDistroInfo(),
    detectVcs(),
  ])
  
  return {
    platform: getHostPlatformForAnalytics(),
    platformRaw: process.env.CLAUDE_CODE_HOST_PLATFORM || process.platform,
    arch: env.arch,
    nodeVersion: env.nodeVersion,
    terminal: envDynamic.terminal,
    packageManagers: packageManagers.join(','),
    runtimes: runtimes.join(','),
    isRunningWithBun: env.isRunningWithBun(),
    isCi: isEnvTruthy(process.env.CI),
    isClaudeCodeRemote: isEnvTruthy(process.env.CLAUDE_CODE_REMOTE),
    version: MACRO.VERSION,
    buildTime: MACRO.BUILD_TIME,
    deploymentEnvironment: env.detectDeploymentEnvironment(),
    ...(getWslVersion() && { wslVersion: getWslVersion() }),
    ...(linuxDistroInfo ?? {}),
    ...(vcs.length > 0 ? { vcs: vcs.join(',') } : {}),
  }
})

/**
 * Convert metadata to 1P event logging format (snake_case)
 */
export function to1PEventFormat(
  metadata: EventMetadata,
  userMetadata: CoreUserData,
  additionalMetadata: Record<string, unknown> = {},
): FirstPartyEventLoggingMetadata {
  const { envContext, processMetrics, ...coreFields } = metadata
  
  // 转换为 proto 兼容格式
  const env: EnvironmentMetadata = {
    platform: envContext.platform,
    platform_raw: envContext.platformRaw,
    arch: envContext.arch,
    node_version: envContext.nodeVersion,
    terminal: envContext.terminal || 'unknown',
    package_managers: envContext.packageManagers,
    runtimes: envContext.runtimes,
    is_running_with_bun: envContext.isRunningWithBun,
    is_ci: envContext.isCi,
    version: envContext.version,
    build_time: envContext.buildTime,
    deployment_environment: envContext.deploymentEnvironment,
  }
  
  const core: FirstPartyEventLoggingCoreMetadata = {
    session_id: coreFields.sessionId,
    model: coreFields.model,
    user_type: coreFields.userType,
    is_interactive: coreFields.isInteractive === 'true',
    client_type: coreFields.clientType,
  }
  
  return {
    env,
    ...(processMetrics && {
      process: Buffer.from(jsonStringify(processMetrics)).toString('base64'),
    }),
    core,
    additional: {
      ...(metadata.rh && { rh: metadata.rh }),
      ...additionalMetadata,
    },
  }
}
```

### 3.6 sink.ts - 分析路由核心

**文件路径**: `services/analytics/sink.ts`

这是**分析路由模块**，将事件分发到 Datadog 和 1P 日志系统：

```typescript
import { trackDatadogEvent } from './datadog.js'
import { logEventTo1P, shouldSampleEvent } from './firstPartyEventLogger.js'
import { checkStatsigFeatureGate_CACHED_MAY_BE_STALE } from './growthbook.js'
import { attachAnalyticsSink, stripProtoFields } from './index.js'
import { isSinkKilled } from './sinkKillswitch.js'

const DATADOG_GATE_NAME = 'tengu_log_datadog_events'

// Datadog 开关状态（启动时初始化）
let isDatadogGateEnabled: boolean | undefined = undefined

/**
 * Check if Datadog tracking is enabled
 */
function shouldTrackDatadog(): boolean {
  if (isSinkKilled('datadog')) return false
  if (isDatadogGateEnabled !== undefined) {
    return isDatadogGateEnabled
  }
  // 回退到缓存值
  return checkStatsigFeatureGate_CACHED_MAY_BE_STALE(DATADOG_GATE_NAME)
}

/**
 * Log event implementation (synchronous)
 */
function logEventImpl(eventName: string, metadata: LogEventMetadata): void {
  // 采样检查
  const sampleResult = shouldSampleEvent(eventName)
  if (sampleResult === 0) return  // 被采样丢弃
  
  // 添加采样率元数据
  const metadataWithSampleRate = sampleResult !== null
    ? { ...metadata, sample_rate: sampleResult }
    : metadata
  
  // Datadog：剥离 _PROTO_* PII 字段
  if (shouldTrackDatadog()) {
    void trackDatadogEvent(eventName, stripProtoFields(metadataWithSampleRate))
  }
  
  // 1P：保留完整 payload（包括 PII）
  logEventTo1P(eventName, metadataWithSampleRate)
}

/**
 * Initialize analytics sink during app startup
 */
export function initializeAnalyticsSink(): void {
  attachAnalyticsSink({
    logEvent: logEventImpl,
    logEventAsync: logEventAsyncImpl,
  })
}

/**
 * Initialize analytics gates (called during setupBackend)
 */
export function initializeAnalyticsGates(): void {
  isDatadogGateEnabled =
    checkStatsigFeatureGate_CACHED_MAY_BE_STALE(DATADOG_GATE_NAME)
}
```

---

## 4. 遥测数据收集

### 4.1 telemetryAttributes.ts 分析

**文件路径**: `utils/telemetryAttributes.ts`

```typescript
// 基数控制配置
const METRICS_CARDINALITY_DEFAULTS = {
  OTEL_METRICS_INCLUDE_SESSION_ID: true,
  OTEL_METRICS_INCLUDE_VERSION: false,
  OTEL_METRICS_INCLUDE_ACCOUNT_UUID: true,
}

/**
 * Get telemetry attributes for OpenTelemetry
 */
export function getTelemetryAttributes(): Attributes {
  const userId = getOrCreateUserID()
  const sessionId = getSessionId()
  
  const attributes: Attributes = {
    'user.id': userId,
  }
  
  // 可配置的属性包含
  if (shouldIncludeAttribute('OTEL_METRICS_INCLUDE_SESSION_ID')) {
    attributes['session.id'] = sessionId
  }
  if (shouldIncludeAttribute('OTEL_METRICS_INCLUDE_VERSION')) {
    attributes['app.version'] = MACRO.VERSION
  }
  
  // OAuth 账户数据
  const oauthAccount = getOauthAccountInfo()
  if (oauthAccount) {
    const orgId = oauthAccount.organizationUuid
    const email = oauthAccount.emailAddress
    const accountUuid = oauthAccount.accountUuid
    
    if (orgId) attributes['organization.id'] = orgId
    if (email) attributes['user.email'] = email
    
    if (accountUuid && shouldIncludeAttribute('OTEL_METRICS_INCLUDE_ACCOUNT_UUID')) {
      attributes['user.account_uuid'] = accountUuid
      attributes['user.account_id'] =
        process.env.CLAUDE_CODE_ACCOUNT_TAGGED_ID ||
        toTaggedId('user', accountUuid)
    }
  }
  
  // 终端类型
  if (envDynamic.terminal) {
    attributes['terminal.type'] = envDynamic.terminal
  }
  
  return attributes
}
```

### 4.2 收集的数据点

| 数据类别 | 字段 | 来源 | 说明 |
|----------|------|------|------|
| **用户标识** | `user.id` | `getOrCreateUserID()` | 设备级 UUID |
| **会话标识** | `session.id` | `getSessionId()` | 会话 UUID |
| **账户信息** | `user.account_uuid` | OAuth token | 用户账户 UUID |
| **组织信息** | `organization.id` | OAuth token | 组织 UUID |
| **环境信息** | `terminal.type` | 环境变量 | 终端类型 |
| **版本信息** | `app.version` | 编译宏 | 应用版本 |

### 4.3 用户 ID/会话 ID 处理

```typescript
// 用户 ID 创建逻辑 (utils/config.ts)
export function getOrCreateUserID(): string {
  const globalConfig = getGlobalConfig()
  
  if (globalConfig.userID) {
    return globalConfig.userID
  }
  
  // 创建新用户 ID
  const userID = randomUUID()
  saveGlobalConfig({ ...globalConfig, userID })
  return userID
}

// 会话 ID 管理 (bootstrap/state.ts)
export function getSessionId(): string {
  return STATE.sessionId
}

// 会话初始化
export function switchSession(newSessionId: string, newProjectDir: string | null): void {
  STATE.sessionId = newSessionId
  STATE.projectDir = newProjectDir || STATE.projectDir
  // 重置会话级状态
  resetCostState()
}
```

---

## 5. 成本追踪系统

### 5.1 cost-tracker.ts 逐行分析

**文件路径**: `cost-tracker.ts`

```typescript
import {
  addToTotalCostState,
  getCostCounter,
  getTokenCounter,
  getTotalCostUSD,
  getTotalDuration,
  getTotalInputTokens,
  getTotalOutputTokens,
  resetCostState,
} from './bootstrap/state.js'

// 导出公共 API
export {
  getTotalCostUSD as getTotalCost,
  getTotalDuration,
  getTotalInputTokens,
  getTotalOutputTokens,
  resetCostState,
}

/**
 * Stored cost state for session persistence
 */
type StoredCostState = {
  totalCostUSD: number
  totalAPIDuration: number
  totalAPIDurationWithoutRetries: number
  totalToolDuration: number
  totalLinesAdded: number
  totalLinesRemoved: number
  lastDuration: number | undefined
  modelUsage: { [modelName: string]: ModelUsage } | undefined
}

/**
 * Get stored costs from project config
 * 用于会话恢复时读取上次保存的成本
 */
export function getStoredSessionCosts(sessionId: string): StoredCostState | undefined {
  const projectConfig = getCurrentProjectConfig()
  
  // 仅返回匹配会话 ID 的成本
  if (projectConfig.lastSessionId !== sessionId) {
    return undefined
  }
  
  // 构建模型使用统计（含上下文窗口）
  let modelUsage: { [modelName: string]: ModelUsage } | undefined
  if (projectConfig.lastModelUsage) {
    modelUsage = Object.fromEntries(
      Object.entries(projectConfig.lastModelUsage).map(([model, usage]) => [
        model,
        {
          ...usage,
          contextWindow: getContextWindowForModel(model, getSdkBetas()),
          maxOutputTokens: getModelMaxOutputTokens(model).default,
        },
      ]),
    )
  }
  
  return {
    totalCostUSD: projectConfig.lastCost ?? 0,
    totalAPIDuration: projectConfig.lastAPIDuration ?? 0,
    // ... 其他字段
  }
}

/**
 * Save current session costs to project config
 * 用于切换会话前保存累积成本
 */
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

/**
 * Format cost for display
 */
function formatCost(cost: number, maxDecimalPlaces: number = 4): string {
  return `$${cost > 0.5 ? round(cost, 100).toFixed(2) : cost.toFixed(maxDecimalPlaces)}`
}

/**
 * Format model usage breakdown
 */
function formatModelUsage(): string {
  const modelUsageMap = getModelUsage()
  if (Object.keys(modelUsageMap).length === 0) {
    return 'Usage: 0 input, 0 output, 0 cache read, 0 cache write'
  }
  
  // 按短名称累积使用量
  const usageByShortName: { [shortName: string]: ModelUsage } = {}
  for (const [model, usage] of Object.entries(modelUsageMap)) {
    const shortName = getCanonicalName(model)
    // 累加各维度
    accumulated.inputTokens += usage.inputTokens
    accumulated.outputTokens += usage.outputTokens
    // ...
  }
  
  let result = 'Usage by model:'
  for (const [shortName, usage] of Object.entries(usageByShortName)) {
    result += `\n${shortName}: ${formatNumber(usage.inputTokens)} input, ...`
  }
  return result
}

/**
 * Format total cost summary for display
 */
export function formatTotalCost(): string {
  const costDisplay =
    formatCost(getTotalCostUSD()) +
    (hasUnknownModelCost()
      ? ' (costs may be inaccurate due to usage of unknown models)'
      : '')
  
  const modelUsageDisplay = formatModelUsage()
  
  return chalk.dim(
    `Total cost: ${costDisplay}\n` +
    `Total duration (API): ${formatDuration(getTotalAPIDuration())}\n` +
    `Total duration (wall): ${formatDuration(getTotalDuration())}\n` +
    `Total code changes: ${getTotalLinesAdded()} lines added, ${getTotalLinesRemoved()} lines removed\n` +
    `${modelUsageDisplay}`,
  )
}

/**
 * Add to total session cost
 * 核心：每次 API 响应后调用，累积成本和 Token
 */
export function addToTotalSessionCost(
  cost: number,
  usage: Usage,
  model: string,
): number {
  // 累积模型使用量
  const modelUsage = addToTotalModelUsage(cost, usage, model)
  addToTotalCostState(cost, modelUsage, model)
  
  // 记录到 OTel counter
  const attrs = isFastModeEnabled() && usage.speed === 'fast'
    ? { model, speed: 'fast' }
    : { model }
  
  getCostCounter()?.add(cost, attrs)
  getTokenCounter()?.add(usage.input_tokens, { ...attrs, type: 'input' })
  getTokenCounter()?.add(usage.output_tokens, { ...attrs, type: 'output' })
  getTokenCounter()?.add(usage.cache_read_input_tokens ?? 0, { ...attrs, type: 'cacheRead' })
  getTokenCounter()?.add(usage.cache_creation_input_tokens ?? 0, { ...attrs, type: 'cacheCreation' })
  
  // 处理 advisor 工具成本
  let totalCost = cost
  for (const advisorUsage of getAdvisorUsage(usage)) {
    const advisorCost = calculateUSDCost(advisorUsage.model, advisorUsage)
    logEvent('tengu_advisor_tool_token_usage', {
      advisor_model: advisorUsage.model,
      input_tokens: advisorUsage.input_tokens,
      output_tokens: advisorUsage.output_tokens,
      cost_usd_micros: Math.round(advisorCost * 1_000_000),
    })
    totalCost += addToTotalSessionCost(advisorCost, advisorUsage, advisorUsage.model)
  }
  
  return totalCost
}
```

### 5.2 costHook.ts 分析

**文件路径**: `costHook.ts`

```typescript
import { useEffect } from 'react'
import { formatTotalCost, saveCurrentSessionCosts } from './cost-tracker.js'
import { hasConsoleBillingAccess } from './utils/billing.js'

/**
 * React hook for cost summary display on exit
 */
export function useCostSummary(
  getFpsMetrics?: () => FpsMetrics | undefined,
): void {
  useEffect(() => {
    const f = () => {
      // 显示成本摘要（仅订阅用户或 ant）
      if (hasConsoleBillingAccess()) {
        process.stdout.write('\n' + formatTotalCost() + '\n')
      }
      
      // 保存当前会话成本
      saveCurrentSessionCosts(getFpsMetrics?.())
    }
    
    // 注册 exit 事件处理
    process.on('exit', f)
    return () => {
      process.off('exit', f)
    }
  }, [])
}
```

### 5.3 成本显示逻辑

**命令**: `/cost` (`commands/cost/cost.ts`)

```typescript
import { formatTotalCost } from '../../cost-tracker.js'
import { currentLimits } from '../../services/claudeAiLimits.js'
import { isClaudeAISubscriber } from '../../utils/auth.js'

export const call: LocalCommandCall = async () => {
  if (isClaudeAISubscriber()) {
    let value: string
    
    // 检查是否使用超额
    if (currentLimits.isUsingOverage) {
      value = 'You are currently using your overages...'
    } else {
      value = 'You are currently using your subscription...'
    }
    
    // ant 用户额外显示详细成本
    if (process.env.USER_TYPE === 'ant') {
      value += `\n\n[ANT-ONLY] Showing cost anyway:\n ${formatTotalCost()}`
    }
    
    return { type: 'text', value }
  }
  
  // 非订阅用户直接显示成本
  return { type: 'text', value: formatTotalCost() }
}
```

---

## 6. 从零实现指南

### 6.1 如何设计洞察系统

#### 步骤 1：定义数据模型

```typescript
// 1. 定义会话元数据类型
type SessionMeta = {
  session_id: string
  start_time: string
  duration_minutes: number
  tool_counts: Record<string, number>
  input_tokens: number
  output_tokens: number
  // 扩展更多维度...
}

// 2. 定义分析结果类型
type SessionFacets = {
  underlying_goal: string
  goal_categories: Record<string, number>
  outcome: 'fully_achieved' | 'mostly_achieved' | ...
  friction_counts: Record<string, number>
  brief_summary: string
}
```

#### 步骤 2：设计数据收集管道

```typescript
// 从日志文件提取统计
function extractStatsFromLog(log: LogOption): SessionMeta {
  const stats = {
    toolCounts: {},
    inputTokens: 0,
    outputTokens: 0,
  }
  
  for (const msg of log.messages) {
    if (msg.type === 'assistant') {
      // 提取 usage
      const usage = msg.message.usage
      stats.inputTokens += usage?.input_tokens || 0
      
      // 提取工具调用
      for (const block of msg.message.content) {
        if (block.type === 'tool_use') {
          stats.toolCounts[block.name] = (stats.toolCounts[block.name] || 0) + 1
        }
      }
    }
  }
  
  return stats
}
```

### 6.2 遥测数据收集实现

#### 基础架构

```typescript
// 1. Sink 接口设计
type AnalyticsSink = {
  logEvent: (name: string, metadata: Record<string, unknown>) => void
  logEventAsync: (name: string, metadata: Record<string, unknown>) => Promise<void>
}

// 2. 事件队列（启动前暂存）
const eventQueue: QueuedEvent[] = []
let sink: AnalyticsSink | null = null

// 3. 连接 sink
function attachSink(newSink: AnalyticsSink): void {
  sink = newSink
  // 异步排空队列
  queueMicrotask(() => {
    for (const event of eventQueue) {
      sink.logEvent(event.name, event.metadata)
    }
  })
}

// 4. 记录事件
function logEvent(name: string, metadata: Record<string, unknown>): void {
  if (!sink) {
    eventQueue.push({ name, metadata })
    return
  }
  sink.logEvent(name, metadata)
}
```

#### 采样机制

```typescript
// 采样配置（来自远程特性开关）
type SamplingConfig = {
  [eventName: string]: { sample_rate: number }
}

function shouldSample(eventName: string): number | null {
  const config = getSamplingConfig()
  const eventConfig = config[eventName]
  
  if (!eventConfig) return null  // 无配置 = 100%
  
  const rate = eventConfig.sample_rate
  if (rate <= 0) return 0  // 丢弃
  if (rate >= 1) return null  // 全量
  
  // 随机采样
  return Math.random() < rate ? rate : 0
}
```

### 6.3 成本追踪实现

#### 核心状态管理

```typescript
// 全局成本状态
const STATE = {
  totalCostUSD: 0,
  totalAPIDuration: 0,
  modelUsage: {} as Record<string, ModelUsage>,
}

// 累加成本
function addToTotalCost(cost: number, usage: Usage, model: string): void {
  STATE.totalCostUSD += cost
  
  // 累加模型使用量
  const modelUsage = STATE.modelUsage[model] || {
    inputTokens: 0,
    outputTokens: 0,
    costUSD: 0,
  }
  
  modelUsage.inputTokens += usage.input_tokens
  modelUsage.outputTokens += usage.output_tokens
  modelUsage.costUSD += cost
  
  STATE.modelUsage[model] = modelUsage
}

// 格式化显示
function formatCost(): string {
  const cost = STATE.totalCostUSD
  return `$${cost.toFixed(4)}`
}
```

#### 持久化机制

```typescript
// 保存到项目配置
function saveCosts(): void {
  saveProjectConfig({
    lastCost: STATE.totalCostUSD,
    lastModelUsage: STATE.modelUsage,
    lastSessionId: getSessionId(),
  })
}

// 从项目配置恢复
function restoreCosts(sessionId: string): boolean {
  const config = getProjectConfig()
  if (config.lastSessionId !== sessionId) return false
  
  STATE.totalCostUSD = config.lastCost || 0
  STATE.modelUsage = config.lastModelUsage || {}
  return true
}
```

#### API 响应处理

```typescript
// 在 API 响应处理中调用
async function handleApiResponse(response: ApiResponse): void {
  const usage = response.usage
  const model = response.model
  const cost = calculateCost(model, usage)
  
  addToTotalCost(cost, usage, model)
  
  // 记录到遥测
  logEvent('api_response', {
    model,
    input_tokens: usage.input_tokens,
    output_tokens: usage.output_tokens,
    cost_usd_micros: Math.round(cost * 1_000_000),
  })
}
```

---

## 7. 关键设计模式总结

### 7.1 分层架构

```
┌─────────────────────────────────────────────────────────────┐
│                    commands/insights.ts                      │
│                  (用户命令 + HTML 报告生成)                    │
├─────────────────────────────────────────────────────────────┤
│                 services/analytics/index.ts                  │
│                    (公共 API 入口 + 事件队列)                  │
├──────────────────────┬──────────────────────────────────────┤
│   datadog.ts         │    firstPartyEventLogger.ts          │
│   (日志后端)          │    (1P 事件导出器)                    │
├──────────────────────┴──────────────────────────────────────┤
│                    sink.ts (路由层)                           │
│              (事件分发 + 采样 + killswitch)                   │
├─────────────────────────────────────────────────────────────┤
│                   metadata.ts (元数据收集)                    │
│        (环境上下文 + 用户属性 + 进程指标)                      │
├─────────────────────────────────────────────────────────────┤
│                growthbook.ts (特性开关)                       │
│           (远程配置 + 实验暴露日志)                           │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 数据流

```
用户操作 → logEvent() → 事件队列 → sink.logEvent()
                                    ↓
                      ┌─────────────┴─────────────┐
                      ↓                           ↓
              trackDatadogEvent()          logEventTo1P()
                      ↓                           ↓
               批处理 + 刷新              OTel LoggerProvider
                      ↓                           ↓
              Datadog API               FirstPartyEventLoggingExporter
                                                      ↓
                                           /api/event_logging/batch
```

### 7.3 关键特性

| 特性 | 实现位置 | 说明 |
|------|----------|------|
| **零依赖入口** | `index.ts` | 避免循环导入，事件队列暂存 |
| **PII 保护** | `stripProtoFields()` | 剥离 `_PROTO_*` 字段 |
| **用户基数控制** | `getUserBucket()` | SHA256 哈希映射到 30 桶 |
| **二次方退避** | `FirstPartyEventLoggingExporter` | 失败重试算法 |
| **磁盘持久化** | `getCurrentBatchFilePath()` | 失败事件存储 |
| **killswitch** | `sinkKillswitch.ts` | 远程禁用单个 sink |
| **采样机制** | `shouldSampleEvent()` | 远程采样配置 |

---

*Insights 系统分析完成*