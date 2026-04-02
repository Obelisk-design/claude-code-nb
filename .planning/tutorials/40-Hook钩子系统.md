# Hook钩子系统深度解析

**文档版本**: v1.0
**分析日期**: 2026-04-02
**适用版本**: Claude Code CLI

---

## 目录

1. [系统设计概述](#1-系统设计概述)
2. [类型定义详解](#2-类型定义详解)
3. [Schema验证体系](#3-schema验证体系)
4. [钩子事件体系](#4-钩子事件体系)
5. [钩子执行流程](#5-钩子执行流程)
6. [用户配置指南](#6-用户配置指南)
7. [从零实现案例](#7-从零实现案例)

---

## 1. 系统设计概述

### 1.1 设计理念

Hook系统是Claude Code CLI的核心扩展机制，允许用户在CLI生命周期的关键节点注入自定义逻辑。设计遵循以下原则：

**核心设计目标**：
- **可扩展性**: 通过配置文件定义钩子，无需修改源码
- **安全性**: 沙箱隔离、权限校验、环境变量保护
- **灵活性**: 支持命令、LLM提示、HTTP请求、Agent验证四种钩子类型
- **性能**: 异步执行、超时控制、后台任务管理

**架构分离**：
```
用户配置层 (settings.json)
    ↓
Schema验证层 (schemas/hooks.ts)
    ↓
类型定义层 (types/hooks.ts)
    ↓
执行引擎层 (utils/hooks/)
    ↓
事件系统层 (hookEvents.ts)
```

### 1.2 核心文件定位

| 文件路径 | 功能职责 |
|---------|---------|
| `types/hooks.ts` | 类型定义、结果处理、回调接口 |
| `schemas/hooks.ts` | Zod Schema验证、配置解析 |
| `utils/hooks.ts` | 主执行引擎、流程编排 |
| `utils/hooks/execCommandHook.ts` | Shell命令执行 |
| `utils/hooks/execPromptHook.ts` | LLM提示评估 |
| `utils/hooks/execAgentHook.ts` | Agent多轮验证 |
| `utils/hooks/execHttpHook.ts` | HTTP请求处理 |
| `utils/hooks/hookEvents.ts` | 事件广播系统 |
| `utils/hooks/sessionHooks.ts` | 会话级钩子管理 |
| `utils/hooks/AsyncHookRegistry.ts` | 异步钩子注册表 |

---

## 2. 类型定义详解

### 2.1 核心类型体系 (types/hooks.ts)

#### 2.1.1 HookCallback 回调钩子

**定义位置**: `types/hooks.ts:210-226`

```typescript
export type HookCallback = {
  type: 'callback'
  callback: (
    input: HookInput,
    toolUseID: string | null,
    abort: AbortSignal | undefined,
    hookIndex?: number,
    context?: HookCallbackContext,
  ) => Promise<HookJSONOutput>
  timeout?: number
  internal?: boolean  // 内部钩子标记（排除于遥测统计）
}
```

**设计要点**：
- **回调钩子无法持久化**: 仅用于内部注册，不写入settings.json
- **上下文访问**: 通过`HookCallbackContext`访问AppState和归属状态
- **超时控制**: 默认超时可配置，防止无限等待

**使用场景**：
- 内置分析钩子（会话文件访问统计）
- 权限系统内部回调
- 插件系统的动态注册

#### 2.1.2 HookResult 执行结果

**定义位置**: `types/hooks.ts:260-276`

```typescript
export type HookResult = {
  message?: Message              // 用户可见消息
  systemMessage?: Message        // 系统提醒消息
  blockingError?: HookBlockingError  // 阻断错误
  outcome: 'success' | 'blocking' | 'non_blocking_error' | 'cancelled'
  preventContinuation?: boolean  // 阻止后续执行
  stopReason?: string            // 停止原因说明
  permissionBehavior?: 'ask' | 'deny' | 'allow' | 'passthrough'
  hookPermissionDecisionReason?: string
  additionalContext?: string     // 注入上下文
  initialUserMessage?: string    // SessionStart专用
  updatedInput?: Record<string, unknown>  // PreToolUse修改输入
  updatedMCPToolOutput?: unknown  // PostToolUse修改MCP输出
  permissionRequestResult?: PermissionRequestResult
  retry?: boolean                // PermissionDenied重试标记
}
```

**outcome状态说明**：
- `success`: 钩子成功完成，流程继续
- `blocking`: 钩子阻断流程，显示错误给模型
- `non_blocking_error`: 钩子失败但不阻断，显示错误给用户
- `cancelled`: 钩子被中止（超时/用户取消）

#### 2.1.3 AggregatedHookResult 聚合结果

**定义位置**: `types/hooks.ts:277-290`

```typescript
export type AggregatedHookResult = {
  message?: Message
  blockingErrors?: HookBlockingError[]  // 多钩子阻断错误
  preventContinuation?: boolean
  stopReason?: string
  hookPermissionDecisionReason?: string
  permissionBehavior?: PermissionResult['behavior']
  additionalContexts?: string[]  // 多钩子上下文聚合
  initialUserMessage?: string
  updatedInput?: Record<string, unknown>
  updatedMCPToolOutput?: unknown
  permissionRequestResult?: PermissionRequestResult
  retry?: boolean
}
```

**聚合策略**：
- **阻断错误**: 所有钩子的阻断错误收集为数组
- **上下文合并**: `additionalContexts`数组按钩子执行顺序收集
- **权限决策**: 优先级规则（deny > ask > allow）

#### 2.1.4 HookBlockingError 阻断错误

**定义位置**: `types/hooks.ts:330-333`

```typescript
export interface HookBlockingError {
  blockingError: string  // 错误消息
  command: string        // 执行的命令/提示文本
}
```

**生成时机**：
- 钩子返回`decision: 'block'`
- 钩子退出码为2（阻断协议）
- Agent/Prompt钩子条件不满足

### 2.2 Hook输入类型 (agentSdkTypes.js)

**所有钩子事件都有对应的输入类型**：

```typescript
// PreToolUse钩子输入
type PreToolUseHookInput = {
  tool_name: string
  tool_input: Record<string, unknown>
  tool_use_id: string
} & BaseHookInput

// PostToolUse钩子输入
type PostToolUseHookInput = {
  tool_name: string
  tool_input: Record<string, unknown>
  tool_use_id: string
  response: unknown  // 工具执行结果
} & BaseHookInput

// SessionStart钩子输入
type SessionStartHookInput = {
  source: 'startup' | 'resume' | 'clear' | 'compact'
} & BaseHookInput
```

**BaseHookInput公共字段**：
```typescript
type BaseHookInput = {
  session_id: string      // 会话唯一标识
  transcript_path: string // transcript.json路径
  cwd: string            // 当前工作目录
  permission_mode?: string
  agent_id?: string      // 子代理标识
  agent_type?: string    // 代理类型
}
```

---

## 3. Schema验证体系

### 3.1 核心Schema定义 (schemas/hooks.ts)

#### 3.1.1 HookCommandSchema 联合类型

**定义位置**: `schemas/hooks.ts:176-189`

```typescript
export const HookCommandSchema = lazySchema(() => {
  const { BashCommandHookSchema, PromptHookSchema, AgentHookSchema, HttpHookSchema } = 
    buildHookSchemas()
  return z.discriminatedUnion('type', [
    BashCommandHookSchema,
    PromptHookSchema,
    AgentHookSchema,
    HttpHookSchema,
  ])
})
```

**discriminatedUnion优势**：
- **类型安全**: 根据`type`字段自动推断具体类型
- **验证效率**: 只验证匹配的Schema分支
- **错误清晰**: 类型不匹配时精确报错

#### 3.1.2 命令钩子Schema

**定义位置**: `schemas/hooks.ts:32-65`

```typescript
const BashCommandHookSchema = z.object({
  type: z.literal('command').describe('Shell命令钩子类型'),
  command: z.string().describe('要执行的Shell命令'),
  if: IfConditionSchema(),  // 条件过滤
  shell: z.enum(SHELL_TYPES).optional()
    .describe("Shell解释器: 'bash'使用$SHELL, 'powershell'使用pwsh"),
  timeout: z.number().positive().optional()
    .describe('超时秒数'),
  statusMessage: z.string().optional()
    .describe('自定义spinner状态消息'),
  once: z.boolean().optional()
    .describe('执行一次后移除'),
  async: z.boolean().optional()
    .describe('后台执行不阻塞'),
  asyncRewake: z.boolean().optional()
    .describe('后台执行，退出码2时唤醒模型'),
})
```

**关键字段详解**：

**`if`条件过滤** (`schemas/hooks.ts:17-27`)：
```typescript
const IfConditionSchema = lazySchema(() =>
  z.string().optional().describe(
    '权限规则语法过滤钩子触发 (例如 "Bash(git *)")'
  )
)
```
- 使用权限规则语法匹配工具调用
- 避免不匹配的工具触发钩子
- 提升性能，减少不必要的进程创建

**`asyncRewake`机制**：
- 钩子后台运行，不阻塞主流程
- 退出码2时注入`task-notification`消息唤醒模型
- 用于长时间监控任务（如构建完成通知）

#### 3.1.3 提示钩子Schema

**定义位置**: `schemas/hooks.ts:67-95`

```typescript
const PromptHookSchema = z.object({
  type: z.literal('prompt').describe('LLM提示钩子类型'),
  prompt: z.string().describe('LLM评估提示，使用$ARGUMENTS占位符'),
  if: IfConditionSchema(),
  timeout: z.number().positive().optional(),
  model: z.string().optional()
    .describe('指定模型，默认使用快速小模型'),
  statusMessage: z.string().optional(),
  once: z.boolean().optional(),
})
```

**执行流程**：
1. 替换`$ARGUMENTS`为钩子输入JSON
2. 调用指定模型（默认Haiku）
3. 解析响应JSON：`{ok: true}`或`{ok: false, reason: "..."}`
4. 验证失败时阻断流程

#### 3.1.4 HTTP钩子Schema

**定义位置**: `schemas/hooks.ts:97-126`

```typescript
const HttpHookSchema = z.object({
  type: z.literal('http').describe('HTTP钩子类型'),
  url: z.string().url().describe('POST目标URL'),
  if: IfConditionSchema(),
  timeout: z.number().positive().optional(),
  headers: z.record(z.string(), z.string()).optional()
    .describe('自定义请求头，支持$VAR_NAME环境变量插值'),
  allowedEnvVars: z.array(z.string()).optional()
    .describe('允许插值的环境变量白名单'),
  statusMessage: z.string().optional(),
  once: z.boolean().optional(),
})
```

**安全机制**：

**环境变量插值** (`utils/hooks/execHttpHook.ts:89-108`)：
```typescript
function interpolateEnvVars(
  value: string,
  allowedEnvVars: ReadonlySet<string>,
): string {
  const interpolated = value.replace(
    /\$\{([A-Z_][A-Z0-9_]*)\}|\$([A-Z_][A-Z0-9_]*)/g,
    (_, braced, unbraced) => {
      const varName = braced ?? unbraced
      if (!allowedEnvVars.has(varName)) {
        logForDebugging(`env var $${varName}不在白名单，跳过插值`)
        return ''
      }
      return process.env[varName] ?? ''
    }
  )
  return sanitizeHeaderValue(interpolated)  // 剥离CR/LF/NUL防止注入
}
```

**URL白名单** (`utils/hooks/execHttpHook.ts:48-58`)：
```typescript
function getHttpHookPolicy(): {
  allowedUrls: string[] | undefined
  allowedEnvVars: string[] | undefined
} {
  const settings = settingsModule.getInitialSettings()
  return {
    allowedUrls: settings.allowedHttpHookUrls,
    allowedEnvVars: settings.httpHookAllowedEnvVars,
  }
}
```

#### 3.1.5 Agent钩子Schema

**定义位置**: `schemas/hooks.ts:128-163`

```typescript
const AgentHookSchema = z.object({
  type: z.literal('agent').describe('Agent验证钩子类型'),
  prompt: z.string().describe('验证提示，使用$ARGUMENTS占位符'),
  if: IfConditionSchema(),
  timeout: z.number().positive().optional()
    .describe('Agent执行超时，默认60秒'),
  model: z.string().optional()
    .describe('指定模型，默认Haiku'),
  statusMessage: z.string().optional(),
  once: z.boolean().optional(),
})
```

**Agent钩子特点**：
- **多轮执行**: Agent可调用工具验证条件
- **结构化输出**: 强制使用`SyntheticOutputTool`返回结果
- **最大轮数限制**: 默认50轮防止无限循环
- **权限隔离**: Agent在独立权限模式下运行

### 3.2 HookMatcher配置Schema

**定义位置**: `schemas/hooks.ts:194-204`

```typescript
export const HookMatcherSchema = lazySchema(() =>
  z.object({
    matcher: z.string().optional()
      .describe('匹配模式（如工具名"Write")'),
    hooks: z.array(HookCommandSchema())
      .describe('匹配时执行的钩子列表'),
  })
)
```

**匹配器语义**：
- **matcher字段**: 指定工具名/事件类型等过滤条件
- **hooks数组**: 同一matcher可绑定多个钩子，按顺序执行
- **空matcher**: 匹配所有触发事件

### 3.3 HooksSettings顶层配置

**定义位置**: `schemas/hooks.ts:211-213`

```typescript
export const HooksSchema = lazySchema(() =>
  z.partialRecord(z.enum(HOOK_EVENTS), z.array(HookMatcherSchema()))
)

export type HooksSettings = Partial<Record<HookEvent, HookMatcher[]>>
```

**配置结构示例**：
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {"type": "command", "command": "pre-bash-check.sh"}
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {"type": "command", "command": "post-write-validate.sh"},
          {"type": "prompt", "prompt": "验证文件内容合规性: $ARGUMENTS"}
        ]
      }
    ]
  }
}
```

---

## 4. 钩子事件体系

### 4.1 事件类型定义 (entrypoints/sdk/coreTypes.ts)

**完整事件列表** (`coreTypes.ts:25-53`)：
```typescript
export const HOOK_EVENTS = [
  'PreToolUse',          // 工具执行前
  'PostToolUse',         // 工具执行后
  'PostToolUseFailure',  // 工具执行失败后
  'Notification',        // 通知发送时
  'UserPromptSubmit',    // 用户提交提示时
  'SessionStart',        // 会话启动时
  'SessionEnd',          // 会话结束时
  'Stop',                // 模型响应结束前
  'StopFailure',         // API错误导致停止
  'SubagentStart',       // 子代理启动时
  'SubagentStop',        // 子代理停止时
  'PreCompact',          // 会话压缩前
  'PostCompact',         // 会话压缩后
  'PermissionRequest',   // 权限对话框显示时
  'PermissionDenied',    // 自动模式拒绝工具时
  'Setup',               // 仓库初始化/维护时
  'TeammateIdle',        // Teammate即将空闲时
  'TaskCreated',         // 任务创建时
  'TaskCompleted',       // 任务完成时
  'Elicitation',         // MCP服务器请求用户输入
  'ElicitationResult',   // 用户响应MCP请求后
  'ConfigChange',        // 配置文件变更时
  'WorktreeCreate',      // 创建工作树时
  'WorktreeRemove',      // 移除工作树时
  'InstructionsLoaded',  // 指令文件加载时
  'CwdChanged',          // 工作目录变更时
  'FileChanged',         // 监控文件变更时
] as const
```

### 4.2 事件元数据 (utils/hooks/hooksConfigManager.ts)

**元数据结构** (`hooksConfigManager.ts:26-27`)：
```typescript
export type HookEventMetadata = {
  summary: string          // 事件简短描述
  description: string      // 详细说明和退出码语义
  matcherMetadata?: MatcherMetadata  // 匹配器字段说明
}

export type MatcherMetadata = {
  fieldToMatch: string     // 输入中的匹配字段
  values: string[]         // 可选值列表
}
```

**关键事件详解**：

#### PreToolUse事件

**元数据** (`hooksConfigManager.ts:29-37`)：
```typescript
PreToolUse: {
  summary: '工具执行前',
  description: '输入为工具调用参数JSON。\n退出码0 - 不显示输出\n退出码2 - 显示stderr给模型并阻断工具调用\n其他退出码 - 显示stderr给用户但继续执行',
  matcherMetadata: {
    fieldToMatch: 'tool_name',
    values: toolNames,  // 所有可用工具名
  }
}
```

**核心能力**：
- **阻断工具**: 退出码2阻止工具执行
- **修改输入**: 返回`updatedInput`替换原始参数
- **注入上下文**: `additionalContext`添加提示信息

#### PostToolUse事件

**元数据** (`hooksConfigManager.ts:38-46`)：
```typescript
PostToolUse: {
  summary: '工具执行后',
  description: '输入包含inputs和response字段。\n退出码0 - stdout显示在transcript模式\n退出码2 - stderr立即显示给模型\n其他退出码 - stderr仅显示给用户',
  matcherMetadata: {
    fieldToMatch: 'tool_name',
    values: toolNames,
  }
}
```

**核心能力**：
- **修改MCP输出**: `updatedMCPToolOutput`替换MCP工具返回
- **上下文注入**: 添加执行后分析信息

#### SessionStart事件

**元数据** (`hooksConfigManager.ts:87-94`)：
```typescript
SessionStart: {
  summary: '会话启动时',
  description: '输入包含source字段。\n退出码0 - stdout显示给Claude\n阻断错误被忽略\n其他退出码 - stderr显示给用户',
  matcherMetadata: {
    fieldToMatch: 'source',
    values: ['startup', 'resume', 'clear', 'compact'],
  }
}
```

**特殊行为**：
- **阻断错误忽略**: SessionStart钩子无法阻断会话启动
- **初始消息**: `initialUserMessage`可设置首条用户消息
- **路径监控**: `watchPaths`注册FileChanged监控

#### PermissionRequest事件

**元数据** (`hooksConfigManager.ts:163-172`)：
```typescript
PermissionRequest: {
  summary: '权限对话框显示时',
  description: '输入包含tool_name/tool_input/tool_use_id。\n输出JSON包含决策（allow/deny）。\n退出码0 - 使用钩子决策',
  matcherMetadata: {
    fieldToMatch: 'tool_name',
    values: toolNames,
  }
}
```

**决策输出格式**：
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {
      "behavior": "allow",
      "updatedInput": {...},
      "updatedPermissions": [...]
    }
  }
}
```

---

## 5. 钩子执行流程

### 5.1 执行引擎主流程 (utils/hooks.ts)

#### 5.1.1 创建基础输入

**函数定义** (`utils/hooks.ts:301-328`)：
```typescript
export function createBaseHookInput(
  permissionMode?: string,
  sessionId?: string,
  agentInfo?: { agentId?: string; agentType?: string },
): {
  session_id: string
  transcript_path: string
  cwd: string
  permission_mode?: string
  agent_id?: string
  agent_type?: string
} {
  const resolvedSessionId = sessionId ?? getSessionId()
  const resolvedAgentType = agentInfo?.agentType ?? getMainThreadAgentType()
  return {
    session_id: resolvedSessionId,
    transcript_path: getTranscriptPathForSession(resolvedSessionId),
    cwd: getCwd(),
    permission_mode: permissionMode,
    agent_id: agentInfo?.agentId,
    agent_type: resolvedAgentType,
  }
}
```

**字段说明**：
- `transcript_path`: 会话记录文件路径，供钩子分析历史
- `agent_type`: 区分主线程代理和子代理调用

#### 5.1.2 命令钩子执行

**核心函数** (`utils/hooks.ts:747-899`)：
```typescript
async function execCommandHook(
  hook: HookCommand & { type: 'command' },
  hookEvent: HookEvent | 'StatusLine' | 'FileSuggestion',
  hookName: string,
  jsonInput: string,
  signal: AbortSignal,
  hookId: string,
  hookIndex?: number,
  pluginRoot?: string,
  pluginId?: string,
  skillRoot?: string,
  forceSyncExecution?: boolean,
  requestPrompt?: (request: PromptRequest) => Promise<PromptResponse>,
): Promise<{
  stdout: string
  stderr: string
  output: string
  status: number
  aborted?: boolean
  backgrounded?: boolean
}>
```

**Shell选择逻辑** (`utils/hooks.ts:781-811`)：
```typescript
// Windows PowerShell路径：使用原生路径，跳过POSIX转换
const shellType = hook.shell ?? DEFAULT_HOOK_SHELL
const isPowerShell = shellType === 'powershell'

const toHookPath =
  isWindows && !isPowerShell
    ? (p: string) => windowsPathToPosixPath(p)  // Git Bash需要POSIX路径
    : (p: string) => p                          // PowerShell使用原生路径
```

**环境变量注入** (`utils/hooks.ts:882-926`)：
```typescript
const envVars: NodeJS.ProcessEnv = {
  ...subprocessEnv(),
  CLAUDE_PROJECT_DIR: toHookPath(projectDir),
}

// 插件钩子特有变量
if (pluginRoot) {
  envVars.CLAUDE_PLUGIN_ROOT = toHookPath(pluginRoot)
  if (pluginId) {
    envVars.CLAUDE_PLUGIN_DATA = toHookPath(getPluginDataDir(pluginId))
  }
}

// SessionStart/Setup钩子支持环境文件
if (hookEvent === 'SessionStart' || hookEvent === 'Setup' || ...) {
  envVars.CLAUDE_ENV_FILE = await getHookEnvFilePath(hookEvent, hookIndex)
}
```

#### 5.1.3 异步钩子检测

**首行检测协议** (`utils/hooks.ts:1117-1164`)：
```typescript
// 钩子首行输出决定执行模式
if (!initialResponseChecked) {
  const firstLine = firstLineOf(stdout).trim()
  if (!firstLine.includes('}')) return
  initialResponseChecked = true
  
  const parsed = jsonParse(firstLine)
  if (isAsyncHookJSONOutput(parsed) && !forceSyncExecution) {
    const processId = `async_hook_${child.pid}`
    const backgrounded = executeInBackground({
      processId,
      hookId,
      shellCommand,
      asyncResponse: parsed,
      hookEvent,
      hookName,
      command: hook.command,
      pluginId,
    })
    if (backgrounded) {
      shellCommandTransferred = true
      asyncResolve?.({ stdout, stderr, output, status: 0 })
    }
  }
}
```

**异步响应识别** (`types/hooks.ts:182-186`)：
```typescript
export function isAsyncHookJSONOutput(
  json: HookJSONOutput,
): json is AsyncHookJSONOutput {
  return 'async' in json && json.async === true
}
```

#### 5.1.4 提示请求协议

**交互式钩子** (`utils/hooks.ts:1073-1110`)：
```typescript
// 钩子可请求用户交互
if (requestPrompt) {
  lineBuffer += data
  const lines = lineBuffer.split('\n')
  lineBuffer = lines.pop() ?? ''
  
  for (const line of lines) {
    const trimmed = line.trim()
    if (!trimmed) continue
    
    const parsed = jsonParse(trimmed)
    const validation = promptRequestSchema().safeParse(parsed)
    if (validation.success) {
      processedPromptLines.add(trimmed)
      const promptReq = validation.data
      promptChain = promptChain.then(async () => {
        const response = await requestPrompt(promptReq)
        child.stdin.write(jsonStringify(response) + '\n', 'utf8')
      })
    }
  }
}
```

**提示请求格式** (`types/hooks.ts:28-42`)：
```typescript
export const promptRequestSchema = lazySchema(() =>
  z.object({
    prompt: z.string(),    // 请求ID
    message: z.string(),   // 提示消息
    options: z.array(
      z.object({
        key: z.string(),
        label: z.string(),
        description: z.string().optional(),
      }),
    ),
  })
)

export type PromptResponse = {
  prompt_response: string  // 匹配请求ID
  selected: string         // 用户选择的key
}
```

### 5.2 输出处理流程

#### 5.2.1 JSON输出解析

**解析函数** (`utils/hooks.ts:399-451`)：
```typescript
function parseHookOutput(stdout: string): {
  json?: HookJSONOutput
  plainText?: string
  validationError?: string
} {
  const trimmed = stdout.trim()
  
  // 非JSON输出直接返回纯文本
  if (!trimmed.startsWith('{')) {
    return { plainText: stdout }
  }
  
  // JSON验证
  const result = validateHookJson(trimmed)
  if ('json' in result) {
    return result
  }
  
  // 验证失败，返回错误提示
  return {
    plainText: stdout,
    validationError: `${result.validationError}\n\nExpected schema:\n${jsonStringify({
      continue: 'boolean (optional)',
      suppressOutput: 'boolean (optional)',
      decision: '"approve" | "block" (optional)',
      hookSpecificOutput: { ... }
    }, null, 2)}`
  }
}
```

#### 5.2.2 同步响应处理

**处理函数** (`utils/hooks.ts:489-737`)：
```typescript
function processHookJSONOutput({
  json,
  command,
  hookName,
  toolUseID,
  hookEvent,
  expectedHookEvent,
  stdout,
  stderr,
  exitCode,
  durationMs,
}: {...}): Partial<HookResult> {
  const result: Partial<HookResult> = {}
  
  // 处理通用字段
  if (syncJson.continue === false) {
    result.preventContinuation = true
    if (syncJson.stopReason) {
      result.stopReason = syncJson.stopReason
    }
  }
  
  // 处理决策字段
  if (json.decision) {
    switch (json.decision) {
      case 'approve':
        result.permissionBehavior = 'allow'
        break
      case 'block':
        result.permissionBehavior = 'deny'
        result.blockingError = {
          blockingError: json.reason || 'Blocked by hook',
          command,
        }
        break
    }
  }
  
  // 处理hookSpecificOutput（按事件类型分支）
  if (json.hookSpecificOutput) {
    switch (json.hookSpecificOutput.hookEventName) {
      case 'PreToolUse':
        if (json.hookSpecificOutput.permissionDecision) {
          // 处理权限决策
        }
        if (json.hookSpecificOutput.updatedInput) {
          result.updatedInput = json.hookSpecificOutput.updatedInput
        }
        result.additionalContext = json.hookSpecificOutput.additionalContext
        break
      
      case 'PostToolUse':
        result.additionalContext = json.hookSpecificOutput.additionalContext
        if (json.hookSpecificOutput.updatedMCPToolOutput) {
          result.updatedMCPToolOutput = json.hookSpecificOutput.updatedMCPToolOutput
        }
        break
      
      case 'SessionStart':
        result.initialUserMessage = json.hookSpecificOutput.initialUserMessage
        if (json.hookSpecificOutput.watchPaths) {
          result.watchPaths = json.hookSpecificOutput.watchPaths
        }
        break
    }
  }
  
  return result
}
```

### 5.3 异步钩子注册表

**注册函数** (`utils/hooks/AsyncHookRegistry.ts:30-83`)：
```typescript
export function registerPendingAsyncHook({
  processId,
  hookId,
  asyncResponse,
  hookName,
  hookEvent,
  command,
  shellCommand,
  toolName,
  pluginId,
}: {...}): void {
  const timeout = asyncResponse.asyncTimeout || 15000  // 默认15秒
  
  const stopProgressInterval = startHookProgressInterval({
    hookId,
    hookName,
    hookEvent,
    getOutput: async () => {
      const taskOutput = pendingHooks.get(processId)?.shellCommand?.taskOutput
      if (!taskOutput) {
        return { stdout: '', stderr: '', output: '' }
      }
      const stdout = await taskOutput.getStdout()
      const stderr = taskOutput.getStderr()
      return { stdout, stderr, output: stdout + stderr }
    },
  })
  
  pendingHooks.set(processId, {
    processId,
    hookId,
    hookName,
    hookEvent,
    toolName,
    pluginId,
    command,
    startTime: Date.now(),
    timeout,
    responseAttachmentSent: false,
    shellCommand,
    stopProgressInterval,
  })
}
```

**响应轮询** (`AsyncHookRegistry.ts:113-268`)：
```typescript
export async function checkForAsyncHookResponses(): Promise<Array<{
  processId: string
  response: SyncHookJSONOutput
  hookName: string
  hookEvent: HookEvent
  stdout: string
  stderr: string
  exitCode?: number
}>> {
  const responses = []
  const hooks = Array.from(pendingHooks.values())
  
  const settled = await Promise.allSettled(
    hooks.map(async hook => {
      const stdout = await hook.shellCommand?.taskOutput.getStdout()
      const stderr = hook.shellCommand?.taskOutput.getStderr()
      
      if (hook.shellCommand.status !== 'completed') {
        return { type: 'skip' }
      }
      
      // 检查是否有同步响应输出
      const lines = stdout.split('\n')
      const execResult = await hook.shellCommand.result
      const exitCode = execResult.code
      
      let response: SyncHookJSONOutput = {}
      for (const line of lines) {
        if (line.trim().startsWith('{')) {
          const parsed = jsonParse(line.trim())
          if (!('async' in parsed)) {
            response = parsed
            break
          }
        }
      }
      
      hook.responseAttachmentSent = true
      await finalizeHook(hook, exitCode, exitCode === 0 ? 'success' : 'error')
      
      return {
        type: 'response',
        processId: hook.processId,
        payload: {
          processId: hook.processId,
          response,
          hookName: hook.hookName,
          hookEvent: hook.hookEvent,
          stdout,
          stderr,
          exitCode,
        }
      }
    })
  )
  
  return responses
}
```

### 5.4 事件广播系统

**事件类型** (`utils/hooks/hookEvents.ts:23-54`)：
```typescript
export type HookStartedEvent = {
  type: 'started'
  hookId: string
  hookName: string
  hookEvent: string
}

export type HookProgressEvent = {
  type: 'progress'
  hookId: string
  hookName: string
  hookEvent: string
  stdout: string
  stderr: string
  output: string
}

export type HookResponseEvent = {
  type: 'response'
  hookId: string
  hookName: string
  hookEvent: string
  output: string
  stdout: string
  stderr: string
  exitCode?: number
  outcome: 'success' | 'error' | 'cancelled'
}
```

**事件发射** (`hookEvents.ts:93-177`)：
```typescript
export function emitHookStarted(
  hookId: string,
  hookName: string,
  hookEvent: string,
): void {
  if (!shouldEmit(hookEvent)) return
  emit({
    type: 'started',
    hookId,
    hookName,
    hookEvent,
  })
}

export function emitHookResponse(data: {
  hookId: string
  hookName: string
  hookEvent: string
  output: string
  stdout: string
  stderr: string
  exitCode?: number
  outcome: 'success' | 'error' | 'cancelled'
}): void {
  const outputToLog = data.stdout || data.stderr || data.output
  if (outputToLog) {
    logForDebugging(
      `Hook ${data.hookName} (${data.hookEvent}) ${data.outcome}:\n${outputToLog}`
    )
  }
  
  if (!shouldEmit(data.hookEvent)) return
  emit({
    type: 'response',
    ...data,
  })
}
```

---

## 6. 用户配置指南

### 6.1 配置文件位置

| 配置类型 | 文件路径 | 作用域 |
|---------|---------|-------|
| 用户配置 | `~/.claude/settings.json` | 全局 |
| 项目配置 | `.claude/settings.json` | 项目级 |
| 本地配置 | `.claude/settings.local.json` | 本地覆盖 |
| 管理配置 | Managed配置 | 企业策略 |

**配置优先级**：本地配置 > 项目配置 > 用户配置 > 管理配置

### 6.2 基础配置示例

#### 6.2.1 PreToolUse钩子

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "scripts/pre-bash-validate.sh",
            "timeout": 5
          }
        ]
      },
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "prompt",
            "prompt": "验证写入文件不包含敏感信息: $ARGUMENTS。返回JSON: {ok: true}或{ok: false, reason: '...'}",
            "model": "claude-sonnet-4-6"
          }
        ]
      }
    ]
  }
}
```

#### 6.2.2 PostToolUse钩子

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "scripts/post-bash-log.sh",
            "async": true
          }
        ]
      },
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "http",
            "url": "https://api.example.com/code-review",
            "headers": {
              "Authorization": "Bearer $API_TOKEN",
              "X-Request-ID": "$REQUEST_ID"
            },
            "allowedEnvVars": ["API_TOKEN", "REQUEST_ID"],
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

#### 6.2.3 SessionStart钩子

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "scripts/session-init.sh",
            "statusMessage": "初始化会话环境"
          }
        ]
      }
    ]
  }
}
```

### 6.3 高级配置技巧

#### 6.3.1 条件过滤

**使用`if`字段减少不必要的钩子触发**：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "scripts/git-pre-check.sh",
            "if": "Bash(git *)"
          }
        ]
      }
    ]
  }
}
```

**规则语法**：
- `Bash(git *)`: 匹配所有git命令
- `Read(*.ts)`: 匹配所有.ts文件读取
- `Write(src/**/*.js)`: 匹配src目录下的.js文件写入

#### 6.3.2 异步钩子唤醒

**长时间任务监控**：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "scripts/monitor-build.sh",
            "asyncRewake": true,
            "timeout": 300
          }
        ]
      }
    ]
  }
}
```

**工作原理**：
- 钩子后台执行，不阻塞工具流程
- 退出码0：静默完成
- 退出码2：注入`task-notification`唤醒模型继续对话

#### 6.3.3 一次执行钩子

**Setup钩子仅在初始化时执行**：

```json
{
  "hooks": {
    "Setup": [
      {
        "matcher": "init",
        "hooks": [
          {
            "type": "command",
            "command": "scripts/project-init.sh",
            "once": true
          }
        ]
      }
    ]
  }
}
```

#### 6.3.4 Agent验证钩子

**复杂条件验证**：

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "agent",
            "prompt": "验证所有测试通过且代码已提交。读取transcript.json分析执行历史: $ARGUMENTS",
            "model": "claude-sonnet-4-6",
            "timeout": 120
          }
        ]
      }
    ]
  }
}
```

### 6.4 钩子脚本最佳实践

#### 6.4.1 PreToolUse脚本示例

```bash
#!/bin/bash
# pre-bash-validate.sh - 验证Bash命令安全性

# 读取JSON输入
read -r json_input

# 解析工具名和输入
tool_name=$(echo "$json_input" | jq -r '.tool_name')
command=$(echo "$json_input" | jq -r '.tool_input.command')

# 检查危险命令
if echo "$command" | grep -qE '(rm -rf|sudo|chmod 777)'; then
  # 返回阻断响应
  jq -n \
    --arg reason "检测到危险命令模式" \
    '{decision: "block", reason: $reason}'
  exit 2
fi

# 检查需要审批的命令
if echo "$command" | grep -qE '(npm publish|git push)'; then
  # 返回审批请求
  jq -n '{decision: "approve", reason: "命令已验证安全性"}'
  exit 0
fi

# 默认允许
exit 0
```

#### 6.4.2 PostToolUse脚本示例

```bash
#!/bin/bash
# post-write-log.sh - 记录文件写入操作

read -r json_input

tool_input=$(echo "$json_input" | jq -r '.tool_input')
file_path=$(echo "$tool_input" | jq -r '.file_path')

# 记录到日志文件
echo "$(date): 写入文件 $file_path" >> ~/.claude/write-log.txt

# 返回上下文注入
jq -n \
  --arg ctx "文件写入已记录到 ~/.claude/write-log.txt" \
  '{
    hookSpecificOutput: {
      hookEventName: "PostToolUse",
      additionalContext: $ctx
    }
  }'
```

#### 6.4.3 SessionStart脚本示例

```bash
#!/bin/bash
# session-init.sh - 会话初始化

read -r json_input
source=$(echo "$json_input" | jq -r '.source')

# 根据启动源执行不同初始化
case "$source" in
  startup)
    # 检查项目环境
    if [ -f "package.json" ]; then
      # 设置初始用户消息
      jq -n \
        --arg msg "项目检测到Node.js环境，已准备npm命令" \
        '{
          hookSpecificOutput: {
            hookEventName: "SessionStart",
            initialUserMessage: $msg,
            watchPaths: ["/package.json", "/package-lock.json"]
          }
        }'
    fi
    ;;
  resume)
    # 恢复会话时加载状态
    echo "会话恢复，加载上次状态..."
    ;;
esac

exit 0
```

### 6.5 错误处理与调试

#### 6.5.1 常见错误类型

**1. JSON格式错误**：
```
Hook JSON output validation failed:
- hookSpecificOutput.hookEventName: Invalid literal value, expected "PreToolUse"

The hook's output was: {"decision": "approve"}
```

**解决方案**：确保输出JSON符合Schema定义。

**2. 超时错误**：
```
Error executing hook: Hook execution timed out after 30 seconds
```

**解决方案**：增加`timeout`字段或优化脚本性能。

**3. Shell路径错误**（Windows）：
```
Hook "pre-check.sh" has shell: 'powershell' but no PowerShell executable found
```

**解决方案**：安装PowerShell或使用默认bash shell。

#### 6.5.2 调试技巧

**启用调试日志**：
```bash
CLAUDE_DEBUG=1 claude-code
```

**日志输出示例**：
```
Hooks: Processing prompt hook with prompt: 验证文件内容...
Hooks: Model response: {"ok": true}
Hooks: Prompt hook condition was met
```

**测试钩子输出**：
```bash
# 手动测试脚本
echo '{"tool_name":"Bash","tool_input":{"command":"git status"}}' | \
  ./scripts/pre-bash-validate.sh
```

---

## 7. 从零实现案例

### 7.1 实现命令钩子执行器

**核心实现要点**：

```typescript
// 1. 创建基础输入
function createHookInput(event: HookEvent, context: ToolUseContext): string {
  const baseInput = createBaseHookInput(
    context.permissionMode,
    context.sessionId,
    { agentId: context.agentId, agentType: context.agentType }
  )
  
  // 合并事件特定字段
  const eventInput = {
    ...baseInput,
    tool_name: context.tool.name,
    tool_input: context.input,
    tool_use_id: context.toolUseID,
  }
  
  return jsonStringify(eventInput)
}

// 2. 执行命令钩子
async function executeCommandHook(
  hook: BashCommandHook,
  input: string,
  signal: AbortSignal
): Promise<HookResult> {
  const timeout = hook.timeout ? hook.timeout * 1000 : 600000
  
  // 构建环境变量
  const env = {
    ...process.env,
    CLAUDE_PROJECT_DIR: getProjectRoot(),
  }
  
  // Spawn子进程
  const child = spawn(hook.command, [], {
    env,
    shell: true,
    cwd: getCwd(),
  })
  
  // 写入JSON输入
  child.stdin.write(input + '\n')
  child.stdin.end()
  
  // 收集输出
  let stdout = ''
  let stderr = ''
  child.stdout.on('data', data => stdout += data)
  child.stderr.on('data', data => stderr += data)
  
  // 等待完成
  const result = await new Promise<{ code: number }>(resolve => {
    child.on('close', code => resolve({ code }))
  })
  
  // 解析输出
  const parsed = parseHookOutput(stdout)
  
  // 处理结果
  if (result.code === 2) {
    return {
      outcome: 'blocking',
      blockingError: {
        blockingError: stderr || 'Blocked by hook',
        command: hook.command,
      },
      preventContinuation: true,
    }
  }
  
  if ('json' in parsed && parsed.json.hookSpecificOutput) {
    return processHookJSONOutput({
      json: parsed.json,
      command: hook.command,
      hookEvent: 'PreToolUse',
      ...
    })
  }
  
  return { outcome: 'success' }
}

// 3. 聚合多个钩子结果
function aggregateHookResults(results: HookResult[]): AggregatedHookResult {
  const blockingErrors = results
    .filter(r => r.blockingError)
    .map(r => r.blockingError!)
  
  const additionalContexts = results
    .filter(r => r.additionalContext)
    .map(r => r.additionalContext!)
  
  // 权限决策优先级
  const permissionBehaviors = results
    .filter(r => r.permissionBehavior)
    .map(r => r.permissionBehavior!)
  
  const finalBehavior = permissionBehaviors.includes('deny')
    ? 'deny'
    : permissionBehaviors.includes('ask')
      ? 'ask'
      : 'allow'
  
  return {
    blockingErrors,
    additionalContexts,
    permissionBehavior: finalBehavior,
    preventContinuation: results.some(r => r.preventContinuation),
  }
}
```

### 7.2 实现LLM提示钩子

**核心实现要点**：

```typescript
async function executePromptHook(
  hook: PromptHook,
  input: string,
  context: ToolUseContext
): Promise<HookResult> {
  // 1. 替换$ARGUMENTS占位符
  const prompt = hook.prompt.replace('$ARGUMENTS', input)
  
  // 2. 创建用户消息
  const userMessage = createUserMessage({ content: prompt })
  
  // 3. 调用模型
  const response = await queryModelWithoutStreaming({
    messages: [userMessage],
    systemPrompt: asSystemPrompt([
      '你的响应必须是JSON格式：{ok: true} 或 {ok: false, reason: "..."}'
    ]),
    model: hook.model ?? 'claude-3-5-haiku',
    options: {
      outputFormat: {
        type: 'json_schema',
        schema: {
          type: 'object',
          properties: {
            ok: { type: 'boolean' },
            reason: { type: 'string' },
          },
          required: ['ok'],
        },
      },
    },
  })
  
  // 4. 解析响应
  const content = extractTextContent(response.message.content)
  const parsed = jsonParse(content)
  
  // 5. 处理结果
  if (!parsed.ok) {
    return {
      outcome: 'blocking',
      blockingError: {
        blockingError: parsed.reason || 'Prompt hook condition failed',
        command: hook.prompt,
      },
      preventContinuation: true,
      stopReason: parsed.reason,
    }
  }
  
  return { outcome: 'success' }
}
```

### 7.3 实现HTTP钩子

**核心实现要点**：

```typescript
async function executeHttpHook(
  hook: HttpHook,
  input: string
): Promise<HookResult> {
  // 1. 检查URL白名单
  const policy = getHttpHookPolicy()
  if (policy.allowedUrls !== undefined) {
    const matched = policy.allowedUrls.some(p => 
      urlMatchesPattern(hook.url, p)
    )
    if (!matched) {
      return {
        outcome: 'non_blocking_error',
        message: createErrorMessage(`URL不在白名单: ${hook.url}`),
      }
    }
  }
  
  // 2. 构建请求头
  const headers: Record<string, string> = {
    'Content-Type': 'application/json',
  }
  
  if (hook.headers) {
    const allowedVars = new Set(
      hook.allowedEnvVars ?? []
    )
    for (const [name, value] of Object.entries(hook.headers)) {
      headers[name] = interpolateEnvVars(value, allowedVars)
    }
  }
  
  // 3. 发送POST请求
  const response = await axios.post(hook.url, input, {
    headers,
    timeout: hook.timeout ? hook.timeout * 1000 : 600000,
    validateStatus: () => true,  // 接受所有状态码
  })
  
  // 4. 处理响应
  const body = response.data
  if (response.status >= 200 && response.status < 300) {
    // 解析JSON响应
    const parsed = parseHttpHookOutput(body)
    if ('json' in parsed) {
      return processHookJSONOutput({
        json: parsed.json,
        command: hook.url,
        ...
      })
    }
    return { outcome: 'success' }
  }
  
  return {
    outcome: 'non_blocking_error',
    message: createErrorMessage(`HTTP请求失败: ${response.status}`),
  }
}
```

### 7.4 实现Agent验证钩子

**核心实现要点**：

```typescript
async function executeAgentHook(
  hook: AgentHook,
  input: string,
  context: ToolUseContext
): Promise<HookResult> {
  // 1. 替换占位符
  const prompt = hook.prompt.replace('$ARGUMENTS', input)
  const userMessage = createUserMessage({ content: prompt })
  
  // 2. 创建结构化输出工具
  const structuredOutputTool = {
    name: 'ReturnResult',
    inputSchema: z.object({
      ok: z.boolean(),
      reason: z.string().optional(),
    }),
    async prompt() {
      return '使用此工具返回验证结果。必须调用一次。'
    },
  }
  
  // 3. 配置Agent上下文
  const agentContext = {
    ...context,
    options: {
      ...context.options,
      tools: [...context.options.tools, structuredOutputTool],
      mode: 'dontAsk',  // 自动执行模式
    },
  }
  
  // 4. 多轮执行Agent
  let structuredResult: { ok: boolean; reason?: string } | null = null
  const MAX_TURNS = 50
  
  for await (const message of query({
    messages: [userMessage],
    systemPrompt: asSystemPrompt([
      '你正在验证Claude Code的停止条件。',
      `会话transcript路径: ${context.transcriptPath}`,
      '使用可用工具检查代码库验证条件。',
      '验证完成后调用ReturnResult工具返回结果。',
    ]),
    toolUseContext: agentContext,
  })) {
    // 检查结构化输出
    if (
      message.type === 'attachment' &&
      message.attachment.type === 'structured_output'
    ) {
      structuredResult = message.attachment.data
      break
    }
    
    // 检查轮数限制
    if (message.type === 'assistant' && turnCount >= MAX_TURNS) {
      return { outcome: 'cancelled' }
    }
  }
  
  // 5. 处理结果
  if (!structuredResult) {
    return { outcome: 'cancelled' }
  }
  
  if (!structuredResult.ok) {
    return {
      outcome: 'blocking',
      blockingError: {
        blockingError: structuredResult.reason || 'Agent verification failed',
        command: hook.prompt,
      },
    }
  }
  
  return { outcome: 'success' }
}
```

---

## 附录

### A. 钩子事件完整参考

| 事件名 | 触发时机 | 输入字段 | 阻断能力 |
|-------|---------|---------|---------|
| PreToolUse | 工具执行前 | tool_name, tool_input, tool_use_id | 可阻断 |
| PostToolUse | 工具执行后 | tool_name, tool_input, tool_use_id, response | 不可阻断 |
| PostToolUseFailure | 工具失败后 | tool_name, tool_input, tool_use_id, error | 不可阻断 |
| UserPromptSubmit | 用户提交提示 | prompt | 可阻断 |
| SessionStart | 会话启动 | source | 不可阻断 |
| SessionEnd | 会话结束 | reason | 不可阻断 |
| Stop | 响应结束前 | 无 | 可继续对话 |
| StopFailure | API错误停止 | error | 不可阻断 |
| PermissionRequest | 权限对话框 | tool_name, tool_input, tool_use_id | 可决策 |
| Setup | 仓库初始化 | trigger | 不可阻断 |
| PreCompact | 压缩前 | trigger | 可阻断 |
| PostCompact | 压缩后 | trigger, summary | 不可阻断 |
| FileChanged | 文件变更 | file_path, event | 不可阻断 |

### B. 环境变量参考

| 变量名 | 说明 | 适用事件 |
|-------|------|---------|
| CLAUDE_PROJECT_DIR | 项目根目录 | 所有事件 |
| CLAUDE_PLUGIN_ROOT | 插件根目录 | 插件钩子 |
| CLAUDE_PLUGIN_DATA | 插件数据目录 | 插件钩子 |
| CLAUDE_ENV_FILE | 环境文件路径 | SessionStart, Setup, CwdChanged, FileChanged |

### C. 退出码语义

| 退出码 | 语义 | 适用事件 |
|-------|------|---------|
| 0 | 成功 | 所有事件 |
| 1 | 非阻断错误 | 所有事件 |
| 2 | 阻断错误 | PreToolUse, UserPromptSubmit, Stop |
| 其他 | 非阻断错误 | 所有事件 |

---

**文档完成标记**: 本文档基于Claude Code CLI源码逐行分析生成，涵盖Hook系统的完整设计、实现和配置细节。

**版本**: v1.0 | **生成时间**: 2026-04-02 | **源码基准**: Claude Code CLI main分支