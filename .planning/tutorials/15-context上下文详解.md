# 15 - Context上下文详解

**分析日期:** 2026-04-02

## 1. 概述

### Context命令的作用

`/context` 命令是Claude Code CLI中用于可视化当前上下文使用情况的核心命令。它通过彩色网格和详细分类表格，向用户展示：

- 当前上下文窗口的token使用量
- 各类内容（系统提示、工具、消息等）的token占比
- 可用剩余空间
- Memory文件、MCP工具、自定义Agent的token消耗详情

该命令有两种运行模式：
- **交互模式**：显示彩色可视化网格（`commands/context/context.tsx`）
- **非交互模式**：输出Markdown表格格式（`commands/context/context-noninteractive.ts`）

### 上下文注入机制

上下文系统通过两个核心函数实现动态注入：

```
┌─────────────────────────────────────────────────────────────────┐
│                     API请求构建流程                              │
├─────────────────────────────────────────────────────────────────┤
│  1. getSystemPrompt() → 构建系统提示词数组                        │
│  2. getSystemContext() → 获取系统级上下文(git状态等)               │
│  3. getUserContext() → 获取用户级上下文(CLAUDE.md等)               │
│  4. buildEffectiveSystemPrompt() → 合并构建最终系统提示            │
│  5. fetchSystemPromptParts() → 组装缓存安全的上下文部件             │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. commands/context/ 目录逐行分析

### 2.1 commands/context/index.ts 入口文件

**文件路径:** `commands/context/index.ts`

```typescript
// 第1-10行: 导入依赖
import { getIsNonInteractiveSession } from '../../bootstrap/state.js'
import type { Command } from '../../commands.js'

// 第4-10行: 交互式命令定义
export const context: Command = {
  name: 'context',
  description: 'Visualize current context usage as a colored grid',
  isEnabled: () => !getIsNonInteractiveSession(),  // 仅交互模式可用
  type: 'local-jsx',  // JSX组件类型，需要React渲染
  load: () => import('./context.js'),  // 动态加载实现
}

// 第12-24行: 非交互式命令定义
export const contextNonInteractive: Command = {
  type: 'local',
  name: 'context',
  supportsNonInteractive: true,  // 支持非交互模式
  description: 'Show current context usage',
  get isHidden() {
    return !getIsNonInteractiveSession()  // 仅非交互模式可见
  },
  isEnabled() {
    return getIsNonInteractiveSession()  // 仅非交互模式启用
  },
  load: () => import('./context-noninteractive.js'),
}
```

**设计要点：**
- 双命令模式：同一功能在不同场景下有不同的展示形式
- `type: 'local-jsx'` 表示该命令返回React组件，需要Ink渲染
- `isHidden` getter确保命令只在正确模式下显示

### 2.2 commands/context/context.tsx 交互式实现

**文件路径:** `commands/context/context.tsx`

```typescript
// 第1-11行: 导入依赖
import { feature } from 'bun:bundle';
import * as React from 'react';
import type { LocalJSXCommandContext } from '../../commands.js';
import { ContextVisualization } from '../../components/ContextVisualization.js';
import { microcompactMessages } from '../../services/compact/microCompact.js';

// 第18-29行: toApiView函数 - 转换消息为API视角
function toApiView(messages: Message[]): Message[] {
  let view = getMessagesAfterCompactBoundary(messages);
  if (feature('CONTEXT_COLLAPSE')) {
    // 动态加载contextCollapse模块（特性开关）
    const { projectView } = require('../../services/contextCollapse/operations.js')
    view = projectView(view);  // 应用折叠投影
  }
  return view;
}
```

**关键设计：`toApiView`函数的作用**

此函数确保 `/context` 显示的是模型实际看到的内容，而非REPL的原始历史：
1. **移除compact边界前的内容**：`getMessagesAfterCompactBoundary()`
2. **应用context-collapse投影**：将折叠的消息替换为摘要占位符

```typescript
// 第30-63行: call函数 - 命令主入口
export async function call(
  onDone: LocalJSXCommandOnDone,
  context: LocalJSXCommandContext
): Promise<React.ReactNode> {
  const {
    messages,
    getAppState,
    options: { mainLoopModel, tools }
  } = context;

  const apiView = toApiView(messages);  // 转换为API视角

  // 应用microcompact获取准确的API消息表示
  const { messages: compactedMessages } = await microcompactMessages(apiView);

  // 获取终端宽度用于响应式布局
  const terminalWidth = process.stdout.columns || 80;
  const appState = getAppState();

  // 分析上下文使用情况
  const data = await analyzeContextUsage(
    compactedMessages,
    mainLoopModel,
    async () => appState.toolPermissionContext,
    tools,
    appState.agentDefinitions,
    terminalWidth,
    context,  // 完整上下文用于系统提示计算
    undefined,  // mainThreadAgentDefinition
    apiView  // 原始消息用于API使用量提取
  );

  // 渲染为ANSI字符串保留颜色
  const output = await renderToAnsiString(<ContextVisualization data={data} />);
  onDone(output);
  return null;
}
```

### 2.3 commands/context/context-noninteractive.ts 非交互模式

**文件路径:** `commands/context/context-noninteractive.ts`

```typescript
// 第1-15行: 导入和类型定义
import { feature } from 'bun:bundle'
import { microcompactMessages } from '../../services/compact/microCompact.js'
import type { AppState } from '../../state/AppStateStore.js'
import type { Tools, ToolUseContext } from '../../Tool.js'

// 第22-32行: CollectContextDataInput类型
type CollectContextDataInput = {
  messages: Message[]
  getAppState: () => AppState
  options: {
    mainLoopModel: string
    tools: Tools
    agentDefinitions: AgentDefinitionsResult
    customSystemPrompt?: string
    appendSystemPrompt?: string
  }
}

// 第34-77行: collectContextData函数 - 共享数据收集路径
export async function collectContextData(
  context: CollectContextDataInput,
): Promise<ContextData> {
  const { messages, getAppState, options } = context

  // 应用与query.ts相同的预API转换
  let apiView = getMessagesAfterCompactBoundary(messages)
  if (feature('CONTEXT_COLLAPSE')) {
    const { projectView } = require('../../services/contextCollapse/operations.js')
    apiView = projectView(apiView)
  }

  const { messages: compactedMessages } = await microcompactMessages(apiView)
  const appState = getAppState()

  return analyzeContextUsage(
    compactedMessages,
    mainLoopModel,
    async () => appState.toolPermissionContext,
    tools,
    agentDefinitions,
    undefined,  // terminalWidth
    { options: { customSystemPrompt, appendSystemPrompt } },
    undefined,  // mainThreadAgentDefinition
    apiView  // 原始消息用于API使用量提取
  )
}
```

**`collectContextData`与`call`的共享逻辑：**

两个文件都遵循相同的数据收集流程，确保交互和非交互模式的token计数反映模型实际看到的内容。

```typescript
// 第79-88行: call函数 - 返回Markdown表格
export async function call(
  _args: string,
  context: ToolUseContext,
): Promise<{ type: 'text'; value: string }> {
  const data = await collectContextData(context)
  return {
    type: 'text' as const,
    value: formatContextAsMarkdownTable(data),
  }
}

// 第90-325行: formatContextAsMarkdownTable函数
function formatContextAsMarkdownTable(data: ContextData): string {
  // 构建Markdown格式的上下文报告
  // 包含：模型信息、token统计、分类表格、MCP工具、Memory文件等
}
```

---

## 3. context.ts 核心文件逐行分析

**文件路径:** `context.ts`

### 3.1 导入和常量定义 (第1-20行)

```typescript
import { feature } from 'bun:bundle'
import memoize from 'lodash-es/memoize.js'
import {
  getAdditionalDirectoriesForClaudeMd,
  setCachedClaudeMdContent,
} from './bootstrap/state.js'
import { getLocalISODate } from './constants/common.js'

const MAX_STATUS_CHARS = 2000  // git status最大字符数限制

// 系统提示注入（仅用于ant内部调试，缓存破坏）
let systemPromptInjection: string | null = null
```

### 3.2 getSystemPromptInjection 缓存破坏机制 (第25-34行)

```typescript
export function getSystemPromptInjection(): string | null {
  return systemPromptInjection
}

export function setSystemPromptInjection(value: string | null): void {
  systemPromptInjection = value
  // 注入变化时立即清除上下文缓存
  getUserContext.cache.clear?.()
  getSystemContext.cache.clear?.()
}
```

**设计目的：** 用于ant内部的临时调试状态，当需要强制刷新缓存时设置此值。

### 3.3 getGitStatus Git状态获取 (第36-111行)

```typescript
export const getGitStatus = memoize(async (): Promise<string | null> => {
  // 第37-40行: 测试环境跳过
  if (process.env.NODE_ENV === 'test') {
    return null  // 避免测试中的循环依赖
  }

  // 第42-57行: 检查是否为Git仓库
  const startTime = Date.now()
  logForDiagnosticsNoPII('info', 'git_status_started')

  const isGit = await getIsGit()
  if (!isGit) {
    return null  // 非Git仓库跳过
  }

  // 第59-77行: 并行执行多个Git命令
  const [branch, mainBranch, status, log, userName] = await Promise.all([
    getBranch(),
    getDefaultBranch(),
    execFileNoThrow(gitExe(), ['--no-optional-locks', 'status', '--short'])
      .then(({ stdout }) => stdout.trim()),
    execFileNoThrow(gitExe(), ['--no-optional-locks', 'log', '--oneline', '-n', '5'])
      .then(({ stdout }) => stdout.trim()),
    execFileNoThrow(gitExe(), ['config', 'user.name'])
      .then(({ stdout }) => stdout.trim()),
  ])

  // 第84-89行: status截断处理
  const truncatedStatus =
    status.length > MAX_STATUS_CHARS
      ? status.substring(0, MAX_STATUS_CHARS) +
        '\n... (truncated because it exceeds 2k characters...)'
      : status

  // 第96-103行: 构建返回字符串
  return [
    `This is the git status at the start of the conversation...`,
    `Current branch: ${branch}`,
    `Main branch (you will usually use this for PRs): ${mainBranch}`,
    ...(userName ? [`Git user: ${userName}`] : []),
    `Status:\n${truncatedStatus || '(clean)'}`,
    `Recent commits:\n${log}`,
  ].join('\n\n')
})
```

**关键设计：**
- `memoize` 包装确保整个会话只执行一次Git查询
- `--no-optional-locks` 防止锁定Git索引
- 并行执行5个Git命令提高效率
- status超过2000字符自动截断

### 3.4 getSystemContext 系统上下文获取 (第116-150行)

```typescript
export const getSystemContext = memoize(
  async (): Promise<{ [k: string]: string }> => {
    const startTime = Date.now()
    logForDiagnosticsNoPII('info', 'system_context_started')

    // 第124-128行: 获取Git状态（带条件跳过）
    const gitStatus =
      isEnvTruthy(process.env.CLAUDE_CODE_REMOTE) ||
      !shouldIncludeGitInstructions()
        ? null
        : await getGitStatus()

    // 第131-133行: 检查缓存破坏注入
    const injection = feature('BREAK_CACHE_COMMAND')
      ? getSystemPromptInjection()
      : null

    return {
      ...(gitStatus && { gitStatus }),  // 条件添加git状态
      ...(feature('BREAK_CACHE_COMMAND') && injection
        ? { cacheBreaker: `[CACHE_BREAKER: ${injection}]` }
        : {}),
    }
  },
)
```

**返回对象结构：**
```typescript
{
  gitStatus: string | undefined,  // Git仓库状态
  cacheBreaker: string | undefined  // 仅ant内部调试用
}
```

### 3.5 getUserContext 用户上下文获取 (第155-189行)

```typescript
export const getUserContext = memoize(
  async (): Promise<{ [k: string]: string }> => {
    const startTime = Date.now()
    logForDiagnosticsNoPII('info', 'user_context_started')

    // 第165-167行: 检查CLAUDE.md禁用条件
    const shouldDisableClaudeMd =
      isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_CLAUDE_MDS) ||
      (isBareMode() && getAdditionalDirectoriesForClaudeMd().length === 0)

    // 第170-172行: 获取Memory文件
    const claudeMd = shouldDisableClaudeMd
      ? null
      : getClaudeMds(filterInjectedMemoryFiles(await getMemoryFiles()))

    // 第175-176行: 缓存到state
    setCachedClaudeMdContent(claudeMd || null)

    return {
      ...(claudeMd && { claudeMd }),  // CLAUDE.md内容
      currentDate: `Today's date is ${getLocalISODate()}.`,  // 当前日期
    }
  },
)
```

**返回对象结构：**
```typescript
{
  claudeMd: string | undefined,  // 合并后的CLAUDE.md内容
  currentDate: string  // 格式化的当前日期
}
```

---

## 4. 上下文组成部分

### 4.1 系统上下文（git status, cwd）

系统上下文由 `getSystemContext()` 提供，包含：

| 组成部分 | 来源函数 | 说明 |
|---------|---------|------|
| `gitStatus` | `getGitStatus()` | Git仓库状态快照 |
| `cacheBreaker` | `getSystemPromptInjection()` | 仅ant内部调试 |

**注入时机：** 在每个对话开始时注入，通过memoize确保会话期间只计算一次。

### 4.2 用户上下文（CLAUDE.md, memory）

用户上下文由 `getUserContext()` 提供，包含：

| 组成部分 | 来源函数 | 说明 |
|---------|---------|------|
| `claudeMd` | `getClaudeMds()` | 合并后的Memory文件内容 |
| `currentDate` | `getLocalISODate()` | 当前日期字符串 |

**Memory文件加载顺序（claudemd.ts第1-26行注释）：**

```
优先级从低到高（后加载的优先级更高）：
1. Managed memory (/etc/claude-code/CLAUDE.md) - 全局管理指令
2. User memory (~/.claude/CLAUDE.md) - 用户私有全局指令
3. Project memory (CLAUDE.md, .claude/CLAUDE.md, .claude/rules/*.md) - 项目指令
4. Local memory (CLAUDE.local.md) - 私有项目指令
```

### 4.3 工具上下文

工具上下文通过 `analyzeContextUsage()` 计算，包含：

- **系统工具token**：内置工具的定义token
- **MCP工具token**：MCP服务器提供的工具
- **Deferred工具**：延迟加载的工具（工具搜索特性）

### 4.4 MCP上下文

MCP上下文通过 `mcpClients` 参数传入，包含：
- 已连接的MCP服务器列表
- 每个服务器提供的工具schema

---

## 5. utils/queryContext.ts 分析

**文件路径:** `utils/queryContext.ts`

### 5.1 fetchSystemPromptParts() 函数

```typescript
// 第11-29行: 文件用途注释
/**
 * Shared helpers for building the API cache-key prefix (systemPrompt,
 * userContext, systemContext) for query() calls.
 *
 * Lives in its own file because it imports from context.ts and
 * constants/prompts.ts, which are high in the dependency graph.
 */

// 第44-74行: 核心函数实现
export async function fetchSystemPromptParts({
  tools,
  mainLoopModel,
  additionalWorkingDirectories,
  mcpClients,
  customSystemPrompt,
}: {...}): Promise<{
  defaultSystemPrompt: string[]
  userContext: { [k: string]: string }
  systemContext: { [k: string]: string }
}> {
  const [defaultSystemPrompt, userContext, systemContext] = await Promise.all([
    // 自定义提示时跳过默认系统提示构建
    customSystemPrompt !== undefined
      ? Promise.resolve([])
      : getSystemPrompt(tools, mainLoopModel, additionalWorkingDirectories, mcpClients),
    
    getUserContext(),  // 用户上下文始终获取
    
    // 自定义提示时跳过系统上下文
    customSystemPrompt !== undefined ? Promise.resolve({}) : getSystemContext(),
  ])
  return { defaultSystemPrompt, userContext, systemContext }
}
```

**设计要点：**
- 三部分并行获取以提高效率
- 自定义系统提示时跳过默认构建和系统上下文

### 5.2 上下文缓存机制

缓存通过 `lodash-es/memoize` 实现：

```typescript
// context.ts中的memoize应用
export const getSystemContext = memoize(async () => {...})
export const getUserContext = memoize(async () => {...})
export const getGitStatus = memoize(async () => {...})
```

**缓存清除时机：**
1. `setSystemPromptInjection()` 被调用时
2. 会话结束时（新会话自动重新计算）

**缓存策略优势：**
- 避免每个API请求都重新读取文件
- Git状态只查询一次
- CLAUDE.md内容只在会话开始时合并

### 5.3 buildSideQuestionFallbackParams()

```typescript
// 第88-179行: 用于SDK side_question处理
export async function buildSideQuestionFallbackParams({...}): Promise<CacheSafeParams> {
  // 在turn完成前没有stopHooks快照时使用
  // 构建与主循环相同的系统提示前缀以保持缓存命中
  
  const { defaultSystemPrompt, userContext, systemContext } =
    await fetchSystemPromptParts({...})

  const systemPrompt = asSystemPrompt([
    ...(customSystemPrompt !== undefined
      ? [customSystemPrompt]
      : defaultSystemPrompt),
    ...(appendSystemPrompt ? [appendSystemPrompt] : []),
  ])

  // 移除进行中的assistant消息（stop_reason === null）
  const last = messages.at(-1)
  const forkContextMessages =
    last?.type === 'assistant' && last.message.stop_reason === null
      ? messages.slice(0, -1)
      : messages

  return {
    systemPrompt,
    userContext,
    systemContext,
    toolUseContext,
    forkContextMessages,
  }
}
```

---

## 6. analyzeContext.ts 上下文分析核心

**文件路径:** `utils/analyzeContext.ts`

### 6.1 ContextData接口定义

```typescript
// 第190-232行: 返回数据结构
export interface ContextData {
  readonly categories: ContextCategory[]      // 分类列表
  readonly totalTokens: number                // 总token数
  readonly maxTokens: number                  // 最大token数
  readonly rawMaxTokens: number               // 原始最大token数
  readonly percentage: number                 // 使用百分比
  readonly gridRows: GridSquare[][]           // 可视化网格
  readonly model: string                      // 模型名称
  readonly memoryFiles: MemoryFile[]          // Memory文件列表
  readonly mcpTools: McpTool[]                // MCP工具列表
  readonly deferredBuiltinTools?: DeferredBuiltinTool[]  // 延迟内置工具
  readonly systemTools?: SystemToolDetail[]   // 系统工具详情
  readonly systemPromptSections?: SystemPromptSectionDetail[]  // 系统提示分段
  readonly agents: Agent[]                    // 自定义Agent
  readonly slashCommands?: SlashCommandInfo   // Slash命令信息
  readonly skills?: SkillInfo                 // Skills信息
  readonly autoCompactThreshold?: number      // 自动压缩阈值
  readonly isAutoCompactEnabled: boolean      // 是否启用自动压缩
  messageBreakdown?: {...}                    // 消息分解详情
  readonly apiUsage: {...} | null             // API实际使用量
}
```

### 6.2 analyzeContextUsage() 主函数

```typescript
// 第918-998行: 核心分析函数
export async function analyzeContextUsage(
  messages: Message[],
  model: string,
  getToolPermissionContext: () => Promise<ToolPermissionContext>,
  tools: Tools,
  agentDefinitions: AgentDefinitionsResult,
  terminalWidth?: number,
  toolUseContext?: Pick<ToolUseContext, 'options'>,
  mainThreadAgentDefinition?: AgentDefinition,
  originalMessages?: Message[],  // 原始消息用于API使用量提取
): Promise<ContextData> {
  // 第930-935行: 获取模型和上下文窗口
  const runtimeModel = getRuntimeMainLoopModel({...})
  const contextWindow = getContextWindowForModel(runtimeModel, getSdkBetas())

  // 第937-947行: 构建有效系统提示
  const defaultSystemPrompt = await getSystemPrompt(tools, runtimeModel)
  const effectiveSystemPrompt = buildEffectiveSystemPrompt({
    mainThreadAgentDefinition,
    toolUseContext,
    customSystemPrompt,
    defaultSystemPrompt,
    appendSystemPrompt,
  })

  // 第950-983行: 并行计算各类token
  const [
    { systemPromptTokens, systemPromptSections },
    { claudeMdTokens, memoryFileDetails },
    { builtInToolTokens, deferredBuiltinDetails, ... },
    { mcpToolTokens, mcpToolDetails, deferredToolTokens },
    { agentTokens, agentDetails },
    { slashCommandTokens, commandInfo },
    messageBreakdown,
  ] = await Promise.all([...])
}
```

### 6.3 Token计数函数

```typescript
// 第77-109行: 带回退的token计数
async function countTokensWithFallback(
  messages: Anthropic.Beta.Messages.BetaMessageParam[],
  tools: Anthropic.Beta.Messages.BetaToolUnion[],
): Promise<number | null> {
  try {
    const result = await countMessagesTokensWithAPI(messages, tools)
    if (result !== null) return result
  } catch (err) {
    logError(err)
  }

  // 回退到Haiku估算
  try {
    return await countTokensViaHaikuFallback(messages, tools)
  } catch (err) {
    return null
  }
}

// 第234-258行: 工具定义token计数
export async function countToolDefinitionTokens(
  tools: Tools,
  getToolPermissionContext: () => Promise<ToolPermissionContext>,
  agentInfo: AgentDefinitionsResult | null,
  model?: string,
): Promise<number> {
  const toolSchemas = await Promise.all(
    tools.map(tool => toolToAPISchema(tool, {...}))
  )
  const result = await countTokensWithFallback([], toolSchemas)
  return result ?? 0
}
```

### 6.4 分类构建逻辑

```typescript
// 第1007-1097行: 构建上下文分类
const cats: ContextCategory[] = []

// 系统提示始终第一位
if (systemPromptTokens > 0) {
  cats.push({ name: 'System prompt', tokens: systemPromptTokens, color: 'promptBorder' })
}

// 系统工具
if (systemToolsTokens > 0) {
  cats.push({ name: 'System tools', tokens: systemToolsTokens, color: 'inactive' })
}

// MCP工具
if (mcpToolTokens > 0) {
  cats.push({ name: 'MCP tools', tokens: mcpToolTokens, color: 'cyan_FOR_SUBAGENTS_ONLY' })
}

// 延迟工具（不计入实际使用）
if (deferredToolTokens > 0) {
  cats.push({ name: 'MCP tools (deferred)', tokens: deferredToolTokens, isDeferred: true })
}

// Memory文件
if (claudeMdTokens > 0) {
  cats.push({ name: 'Memory files', tokens: claudeMdTokens, color: 'claude' })
}

// 消息
if (messageTokens > 0) {
  cats.push({ name: 'Messages', tokens: messageTokens, color: 'purple_FOR_SUBAGENTS_ONLY' })
}
```

---

## 7. utils/context.ts 上下文窗口配置

**文件路径:** `utils/context.ts`

此文件处理模型上下文窗口大小配置，而非上下文内容构建。

### 7.1 上下文窗口常量

```typescript
// 第8-25行: 核心常量
export const MODEL_CONTEXT_WINDOW_DEFAULT = 200_000  // 默认200k tokens
export const COMPACT_MAX_OUTPUT_TOKENS = 20_000      // 压缩最大输出
const MAX_OUTPUT_TOKENS_DEFAULT = 32_000             // 默认最大输出
const MAX_OUTPUT_TOKENS_UPPER_LIMIT = 64_000         // 上限
export const CAPPED_DEFAULT_MAX_TOKENS = 8_000       // 限制默认值
export const ESCALATED_MAX_TOKENS = 64_000           // 升级后上限
```

### 7.2 1M上下文支持检测

```typescript
// 第31-49行: 1M上下文检测
export function is1mContextDisabled(): boolean {
  return isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_1M_CONTEXT)
}

export function has1mContext(model: string): boolean {
  if (is1mContextDisabled()) return false
  return /\[1m\]/i.test(model)  // 模型名包含[1m]标记
}

export function modelSupports1M(model: string): boolean {
  if (is1mContextDisabled()) return false
  const canonical = getCanonicalName(model)
  return canonical.includes('claude-sonnet-4') || canonical.includes('opus-4-6')
}
```

### 7.3 getContextWindowForModel()

```typescript
// 第51-98行: 获取模型上下文窗口大小
export function getContextWindowForModel(model: string, betas?: string[]): number {
  // ant用户环境变量覆盖
  if (process.env.USER_TYPE === 'ant' && process.env.CLAUDE_CODE_MAX_CONTEXT_TOKENS) {
    const override = parseInt(process.env.CLAUDE_CODE_MAX_CONTEXT_TOKENS, 10)
    if (!isNaN(override) && override > 0) return override
  }

  // [1m]后缀检测
  if (has1mContext(model)) return 1_000_000

  // 模型能力配置
  const cap = getModelCapability(model)
  if (cap?.max_input_tokens && cap.max_input_tokens >= 100_000) {
    return cap.max_input_tokens
  }

  // beta特性检测
  if (betas?.includes(CONTEXT_1M_BETA_HEADER) && modelSupports1M(model)) {
    return 1_000_000
  }

  return MODEL_CONTEXT_WINDOW_DEFAULT
}
```

---

## 8. ContextVisualization组件分析

**文件路径:** `components/ContextVisualization.tsx`

### 8.1 组件结构

```typescript
// 第102-104行: Props定义
interface Props {
  data: ContextData;
}

// 第105-125行: 主组件入口
export function ContextVisualization(t0) {
  const { data } = t0;
  const {
    categories,
    totalTokens,
    rawMaxTokens,
    percentage,
    gridRows,
    model,
    memoryFiles,
    mcpTools,
    deferredBuiltinTools,
    systemTools,
    systemPromptSections,
    agents,
    skills,
    messageBreakdown
  } = data;
}
```

### 8.2 CollapseStatus子组件

```typescript
// 第21-71行: 上下文折叠状态显示
function CollapseStatus() {
  if (feature("CONTEXT_COLLAPSE")) {
    const { getStats, isContextCollapseEnabled } = 
      require("../services/contextCollapse/index.js")
    
    if (!isContextCollapseEnabled()) return null
    
    const s = getStats()
    const parts = []
    
    if (s.collapsedSpans > 0) {
      parts.push(`${s.collapsedSpans} span summarized (${s.collapsedMessages} msgs)`)
    }
    if (s.stagedSpans > 0) {
      parts.push(`${s.stagedSpans} staged`)
    }
    
    return <><Text dimColor>Context strategy: collapse ({summary})</Text>{line2}</>
  }
  return null
}
```

### 8.3 可视化网格渲染

网格通过 `gridRows` 数据渲染，每个格子代表上下文窗口的一部分：

```typescript
// 第161-174行: 网格行渲染
let t11;
if ($[22] !== gridRows) {
  t11 = gridRows.map(_temp5);  // 映射每行为React元素
  $[22] = gridRows;
  $[23] = t11;
}

let t12;
if ($[24] !== t11) {
  t12 = <Box flexDirection="column" flexShrink={0}>{t11}</Box>;
}
```

---

## 9. 从零实现指南

### 9.1 如何设计上下文系统

**核心架构：**

```
┌─────────────────────────────────────────────────────────────────┐
│                     上下文系统架构                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐          │
│  │ 系统上下文   │    │ 用户上下文   │    │ 工具上下文   │          │
│  │ getSystem   │    │ getUser     │    │ analyze     │          │
│  │ Context()   │    │ Context()   │    │ Context()   │          │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘          │
│         │                  │                  │                 │
│         └─────────────────┼──────────────────┘                 │
│                           │                                     │
│                  ┌────────▼────────┐                            │
│                  │ fetchSystem     │                            │
│                  │ PromptParts()   │                            │
│                  └────────┬────────┘                            │
│                           │                                     │
│                  ┌────────▼────────┐                            │
│                  │ buildEffective  │                            │
│                  │ SystemPrompt()  │                            │
│                  └────────┬────────┘                            │
│                           │                                     │
│                  ┌────────▼────────┐                            │
│                  │   API请求构建    │                            │
│                  └─────────────────┘                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 9.2 动态上下文注入实现

**步骤1：定义上下文类型**

```typescript
interface SystemContext {
  gitStatus?: string;
  cwd?: string;
}

interface UserContext {
  claudeMd?: string;
  currentDate: string;
}
```

**步骤2：实现memoized获取函数**

```typescript
import memoize from 'lodash-es/memoize.js';

export const getSystemContext = memoize(async (): Promise<SystemContext> => {
  const gitStatus = await getGitStatus();
  return {
    ...(gitStatus && { gitStatus }),
  };
});

export const getUserContext = memoize(async (): Promise<UserContext> => {
  const claudeMd = await loadMemoryFiles();
  return {
    ...(claudeMd && { claudeMd }),
    currentDate: formatDate(new Date()),
  };
});
```

**步骤3：并行组装上下文**

```typescript
export async function buildContextParts(): Promise<ContextParts> {
  const [systemContext, userContext] = await Promise.all([
    getSystemContext(),
    getUserContext(),
  ]);
  
  return {
    systemContext,
    userContext,
    systemPrompt: buildSystemPrompt(systemContext, userContext),
  };
}
```

### 9.3 缓存策略

**三级缓存架构：**

```
┌─────────────────────────────────────────────────────────────────┐
│                     缓存层级                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Level 1: Memoize缓存（会话级）                                   │
│  - getSystemContext.memoize                                     │
│  - getUserContext.memoize                                       │
│  - getGitStatus.memoize                                         │
│  - 会话期间只计算一次                                             │
│                                                                 │
│  Level 2: ClaudeMd内容缓存                                       │
│  - setCachedClaudeMdContent()                                   │
│  - 供其他模块快速访问                                             │
│                                                                 │
│  Level 3: API缓存命中                                            │
│  - systemPrompt前缀缓存                                          │
│  - userContext缓存                                               │
│  - 通过Anthropic prompt caching实现                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**缓存清除时机：**

```typescript
// 缓存清除函数
export function clearAllContextCaches(): void {
  getSystemContext.cache.clear?.();
  getUserContext.cache.clear?.();
  getGitStatus.cache.clear?.();
  setCachedClaudeMdContent(null);
}
```

### 9.4 实现清单

**最小实现核心文件：**

1. `context.ts` - 系统和用户上下文获取
2. `utils/queryContext.ts` - 上下文部件组装
3. `utils/systemPrompt.ts` - 有效系统提示构建
4. `utils/claudemd.ts` - Memory文件加载

**关键依赖：**

- `lodash-es/memoize.js` - 缓存装饰器
- Git工具函数（`getIsGit`, `getBranch`等）
- 文件系统读取（Memory文件加载）

---

## 10. 总结

### 核心设计原则

1. **分离系统与用户上下文**：系统级（Git状态）和用户级（CLAUDE.md）分开获取
2. **Memoize缓存**：会话期间只计算一次，避免重复IO
3. **并行获取**：`Promise.all`并行执行提高效率
4. **API视角转换**：`toApiView`确保显示内容与模型实际看到的一致
5. **双模式展示**：交互式彩色网格 + 非交互Markdown表格

### 关键文件路径

| 文件 | 路径 | 职责 |
|-----|------|------|
| 入口 | `commands/context/index.ts` | 命令注册 |
| 交互UI | `commands/context/context.tsx` | React可视化 |
| 非交互 | `commands/context/context-noninteractive.ts` | Markdown输出 |
| 核心上下文 | `context.ts` | 系统/用户上下文获取 |
| 上下文组装 | `utils/queryContext.ts` | API缓存前缀构建 |
| Token分析 | `utils/analyzeContext.ts` | 上下文使用量分析 |
| 窗口配置 | `utils/context.ts` | 模型上下文窗口大小 |
| Memory加载 | `utils/claudemd.ts` | CLAUDE.md文件加载 |
| 可视化组件 | `components/ContextVisualization.tsx` | React网格渲染 |

---

*Context系统分析完成 - 2026-04-02*