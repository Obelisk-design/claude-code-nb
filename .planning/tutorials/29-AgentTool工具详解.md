# AgentTool工具详解

**分析日期:** 2026-04-02

---

## 1. 概述

### 1.1 设计目标

AgentTool（工具名为 `Agent`，旧名 `Task`）是Claude Code多代理架构的核心组件，设计目标包括：

**任务委托与并行化**
- 将复杂任务委托给专门的子代理处理
- 支持同步/异步两种执行模式
- 允许主代理在等待子代理完成时继续其他工作

**上下文隔离**
- 子代理拥有独立的对话历史
- 避免主代理上下文膨胀
- 通过Fork机制实现上下文继承

**专业化分工**
- 不同Agent类型针对特定任务优化
- 内置Agent覆盖常见场景（探索、规划、验证等）
- 支持用户自定义Agent

### 1.2 多代理架构优势

```
┌─────────────────────────────────────────────────────────┐
│                    主代理 (Coordinator)                    │
│  ┌─────────────────────────────────────────────────────┐ │
│  │ 对话历史 + 系统提示 + 工具池                           │ │
│  └─────────────────────────────────────────────────────┘ │
│                          │                               │
│          ┌───────────────┼───────────────┐               │
│          ▼               ▼               ▼               │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐     │
│  │  Explore     │ │    Plan      │ │ Verification │     │
│  │  Agent       │ │    Agent     │ │    Agent     │     │
│  │  (只读搜索)   │ │  (架构规划)  │ │  (验证测试)   │     │
│  └──────────────┘ └──────────────┘ └──────────────┘     │
└─────────────────────────────────────────────────────────┘
```

**核心优势：**

| 特性 | 说明 |
|------|------|
| **Token节省** | 子代理结果摘要返回，而非完整对话历史 |
| **并行执行** | 多个子代理可同时运行 |
| **专业优化** | 每个Agent有专门的系统提示和工具集 |
| **后台执行** | 长时间任务可在后台运行，完成后通知 |
| **权限隔离** | 子代理可配置独立的权限模式 |

---

## 2. Agent定义系统

### 2.1 内置Agent列表

AgentTool提供以下内置Agent（定义在 `tools/AgentTool/built-in/` 目录）：

| Agent类型 | 用途 | 模型 | 特性 |
|-----------|------|------|------|
| `general-purpose` | 通用任务处理 | 继承父代理 | 工具: `*` |
| `Explore` | 代码库探索搜索 | haiku/继承 | 只读、省略CLAUDE.md |
| `Plan` | 实现方案规划 | 继承 | 只读、架构设计 |
| `verification` | 实现验证测试 | 继承 | 后台执行、红色标识 |
| `claude-code-guide` | Claude Code帮助文档 | haiku | 权限模式: dontAsk |
| `statusline-setup` | 状态栏配置 | sonnet | 工具: Read, Edit |

**Agent定义结构**（`tools/AgentTool/loadAgentsDir.ts:106-133`）：

```typescript
type BaseAgentDefinition = {
  agentType: string              // Agent类型名称
  whenToUse: string              // 使用场景描述
  tools?: string[]               // 允许的工具列表
  disallowedTools?: string[]     // 禁用的工具列表
  skills?: string[]              // 预加载的技能
  mcpServers?: AgentMcpServerSpec[] // MCP服务器配置
  hooks?: HooksSettings          // 会话钩子
  color?: AgentColorName         // UI显示颜色
  model?: string                 // 模型选择
  effort?: EffortValue           // 努力级别
  permissionMode?: PermissionMode // 权限模式
  maxTurns?: number              // 最大轮次
  background?: boolean           // 是否后台执行
  memory?: AgentMemoryScope      // 持久化内存范围
  isolation?: 'worktree' | 'remote' // 隔离模式
  omitClaudeMd?: boolean         // 是否省略CLAUDE.md
}
```

### 2.2 自定义Agent配置

**通过Markdown文件定义Agent**（`.claude/agents/` 目录）：

```markdown
---
name: code-reviewer
description: Use this agent for code review tasks
tools:
  - Read
  - Grep
  - Glob
disallowedTools:
  - Write
  - Edit
model: sonnet
color: blue
permissionMode: plan
maxTurns: 50
memory: project
---

You are a code review specialist. Your job is to review code changes and provide feedback on:
- Code quality and readability
- Potential bugs or issues
- Security considerations
- Performance implications
```

**Agent配置解析流程**（`tools/AgentTool/loadAgentsDir.ts:541-755`）：

```
.claude/agents/*.md
       │
       ▼
┌──────────────────────────┐
│ parseAgentFromMarkdown() │
│  - 解析frontmatter       │
│  - 验证必需字段          │
│  - 处理工具列表          │
│  - 设置系统提示          │
└──────────────────────────┘
       │
       ▼
CustomAgentDefinition对象
```

**Agent来源优先级**（`tools/AgentTool/agentDisplay.ts:24-32`）：

```typescript
const AGENT_SOURCE_GROUPS = [
  { label: 'User agents', source: 'userSettings' },
  { label: 'Project agents', source: 'projectSettings' },
  { label: 'Local agents', source: 'localSettings' },
  { label: 'Managed agents', source: 'policySettings' },
  { label: 'Plugin agents', source: 'plugin' },
  { label: 'CLI arg agents', source: 'flagSettings' },
  { label: 'Built-in agents', source: 'built-in' },
]
```

### 2.3 Agent继承机制

**模型继承**（`tools/AgentTool/runAgent.ts:340-345`）：

```typescript
const resolvedAgentModel = getAgentModel(
  agentDefinition.model,      // Agent定义的模型
  toolUseContext.options.mainLoopModel,  // 父代理模型
  model,                      // 显式指定的模型
  permissionMode,
)
```

**工具继承规则**（`tools/AgentTool/agentToolUtils.ts:70-116`）：

```typescript
function filterToolsForAgent({
  tools,
  isBuiltIn,
  isAsync,
  permissionMode,
}): Tools {
  return tools.filter(tool => {
    // MCP工具始终允许
    if (tool.name.startsWith('mcp__')) return true
    
    // 全局禁止列表
    if (ALL_AGENT_DISALLOWED_TOOLS.has(tool.name)) return false
    
    // 自定义Agent禁止列表
    if (!isBuiltIn && CUSTOM_AGENT_DISALLOWED_TOOLS.has(tool.name)) return false
    
    // 异步Agent工具限制
    if (isAsync && !ASYNC_AGENT_ALLOWED_TOOLS.has(tool.name)) return false
    
    return true
  })
}
```

**Fork子代理的特殊继承**（`tools/AgentTool/forkSubagent.ts:60-71`）：

```typescript
export const FORK_AGENT = {
  agentType: 'fork',
  tools: ['*'],           // 继承父代理全部工具
  maxTurns: 200,
  model: 'inherit',       // 继承父代理模型
  permissionMode: 'bubble', // 权限提示冒泡到父终端
  source: 'built-in',
  // Fork子代理继承父代理的系统提示，而非重新生成
}
```

---

## 3. 任务分发机制

### 3.1 LocalAgentTask

**同步执行流程**（`tools/AgentTool/AgentTool.tsx:765-1126`）：

```
AgentTool.call()
    │
    ├─► 创建AgentId
    │
    ├─► 设置Agent上下文
    │   runWithAgentContext(syncAgentContext, ...)
    │
    ├─► 包装CWD覆盖
    │   wrapWithCwd(async () => {...})
    │
    ├─► 执行Agent
    │   runAgent({...})
    │
    └─► 返回结果
        finalizeAgentTool(agentMessages, agentId, metadata)
```

**异步执行注册**（`tools/AgentTool/AgentTool.tsx:686-764`）：

```typescript
// 注册异步任务
const agentBackgroundTask = registerAsyncAgent({
  agentId: asyncAgentId,
  description,
  prompt,
  selectedAgent,
  setAppState: rootSetAppState,
  toolUseId: toolUseContext.toolUseId,
})

// 后台执行
void runWithAgentContext(asyncAgentContext, () =>
  wrapWithCwd(() =>
    runAsyncAgentLifecycle({
      taskId: agentBackgroundTask.agentId,
      abortController: agentBackgroundTask.abortController!,
      makeStream: onCacheSafeParams => runAgent({...}),
      metadata,
      ...
    })
  )
)

// 立即返回async_launched状态
return {
  data: {
    status: 'async_launched',
    agentId: agentBackgroundTask.agentId,
    outputFile: getTaskOutputPath(agentBackgroundTask.agentId),
    ...
  }
}
```

### 3.2 RemoteAgentTask

**远程Agent执行**（仅限Ant内部，`tools/AgentTool/AgentTool.tsx:435-482`）：

```typescript
if (effectiveIsolation === 'remote') {
  // 检查远程执行资格
  const eligibility = await checkRemoteAgentEligibility()
  if (!eligibility.eligible) {
    throw new Error(`Cannot launch remote agent:\n${reasons}`)
  }
  
  // 传输到远程CCR环境
  const session = await teleportToRemote({
    initialMessage: prompt,
    description,
    signal: toolUseContext.abortController.signal,
  })
  
  // 注册远程任务
  const { taskId, sessionId } = registerRemoteAgentTask({
    remoteTaskType: 'remote-agent',
    session: { id: session.id, title: session.title },
    ...
  })
  
  return {
    data: {
      status: 'remote_launched',
      taskId,
      sessionUrl: getRemoteTaskSessionUrl(sessionId),
      ...
    }
  }
}
```

### 3.3 调度策略

**执行模式选择**（`tools/AgentTool/AgentTool.tsx:567`）：

```typescript
const shouldRunAsync = (
  run_in_background === true ||        // 用户显式请求
  selectedAgent.background === true || // Agent定义要求后台
  isCoordinator ||                     // 协调器模式
  forceAsync ||                        // Fork实验强制
  assistantForceAsync ||               // Kairos模式
  isProactiveActive()                  // 主动模式
) && !isBackgroundTasksDisabled
```

**Worktree隔离**（`tools/AgentTool/AgentTool.tsx:590-685`）：

```typescript
if (effectiveIsolation === 'worktree') {
  const slug = `agent-${earlyAgentId.slice(0, 8)}`
  worktreeInfo = await createAgentWorktree(slug)
}

// 清理逻辑
const cleanupWorktreeIfNeeded = async () => {
  if (!worktreeInfo) return {}
  
  // 检查是否有变更
  const changed = await hasWorktreeChanges(worktreePath, headCommit)
  if (!changed) {
    // 无变更则自动删除worktree
    await removeAgentWorktree(worktreePath, worktreeBranch, gitRoot)
    return {}
  }
  
  // 有变更则保留worktree
  return { worktreePath, worktreeBranch }
}
```

---

## 4. 工具权限继承

### 4.1 allowedTools配置

**工具解析与验证**（`tools/AgentTool/agentToolUtils.ts:122-225`）：

```typescript
function resolveAgentTools(
  agentDefinition,
  availableTools,
  isAsync,
  isMainThread,
): ResolvedAgentTools {
  // 过滤可用工具
  const filteredAvailableTools = isMainThread
    ? availableTools
    : filterToolsForAgent({
        tools: availableTools,
        isBuiltIn: agentDefinition.source === 'built-in',
        isAsync,
        permissionMode: agentDefinition.permissionMode,
      })
  
  // 处理排除列表
  const disallowedToolSet = new Set(
    disallowedTools?.map(toolSpec => {
      const { toolName } = permissionRuleValueFromString(toolSpec)
      return toolName
    }) ?? []
  )
  
  // 通配符处理
  const hasWildcard = agentTools === undefined || 
    (agentTools.length === 1 && agentTools[0] === '*')
  
  if (hasWildcard) {
    return {
      hasWildcard: true,
      resolvedTools: allowedAvailableTools,
    }
  }
  
  // 显式工具列表解析
  for (const toolSpec of agentTools) {
    const { toolName, ruleContent } = permissionRuleValueFromString(toolSpec)
    
    // 特殊处理Agent工具
    if (toolName === AGENT_TOOL_NAME && ruleContent) {
      allowedAgentTypes = ruleContent.split(',').map(s => s.trim())
    }
    
    // 解析具体工具
    const tool = availableToolMap.get(toolName)
    if (tool) {
      resolved.push(tool)
    } else {
      invalidTools.push(toolSpec)
    }
  }
  
  return { resolvedTools: resolved, allowedAgentTypes }
}
```

### 4.2 权限传递规则

**权限模式继承**（`tools/AgentTool/runAgent.ts:412-463`）：

```typescript
const agentGetAppState = () => {
  const state = toolUseContext.getAppState()
  let toolPermissionContext = state.toolPermissionContext
  
  // Agent定义的权限模式（除非父代理是bypassPermissions/acceptEdits）
  if (
    agentPermissionMode &&
    state.toolPermissionContext.mode !== 'bypassPermissions' &&
    state.toolPermissionContext.mode !== 'acceptEdits'
  ) {
    toolPermissionContext = {
      ...toolPermissionContext,
      mode: agentPermissionMode,
    }
  }
  
  // 异步Agent避免权限提示
  const shouldAvoidPrompts =
    canShowPermissionPrompts !== undefined
      ? !canShowPermissionPrompts
      : agentPermissionMode === 'bubble'
        ? false  // bubble模式冒泡到父终端
        : isAsync
  
  if (shouldAvoidPrompts) {
    toolPermissionContext = {
      ...toolPermissionContext,
      shouldAvoidPermissionPrompts: true,
    }
  }
  
  return { ...state, toolPermissionContext }
}
```

**权限模式说明**：

| 模式 | 行为 |
|------|------|
| `acceptEdits` | 自动接受编辑操作 |
| `plan` | 需要计划审批 |
| `auto` | 自动模式（带分类器检查） |
| `bypassPermissions` | 跳过所有权限检查 |
| `bubble` | 权限提示冒泡到父代理 |
| `dontAsk` | 不询问，自动拒绝 |

---

## 5. 用户使用指南

### 5.1 如何触发多代理

**基本调用方式**：

```typescript
// 调用特定类型Agent
Agent({
  description: "搜索API端点",
  subagent_type: "Explore",
  prompt: "Find all REST API endpoints in the codebase"
})

// 后台执行
Agent({
  description: "后台验证",
  subagent_type: "verification",
  run_in_background: true,
  prompt: "Verify the authentication flow works correctly"
})

// Fork自己（实验性功能）
Agent({
  name: "research-branch",
  description: "分支研究",
  prompt: "Research the database schema changes needed"
})
```

**并行启动多个Agent**：

```typescript
// 在同一消息中使用多个Agent调用
[
  Agent({ description: "探索前端", subagent_type: "Explore", prompt: "..." }),
  Agent({ description: "探索后端", subagent_type: "Explore", prompt: "..." }),
  Agent({ description: "探索测试", subagent_type: "Explore", prompt: "..." })
]
```

### 5.2 /agents命令使用

**查看可用Agent**：

```
/agents
```

输出示例：
```
User agents:
  (无)

Project agents:
  code-reviewer: Code review specialist (Tools: Read, Grep, Glob)

Built-in agents:
  general-purpose: General-purpose agent for complex tasks (Tools: All tools)
  Explore: Fast codebase exploration (Tools: Bash, Glob, Grep, Read)
  Plan: Implementation planning (Tools: Bash, Glob, Grep, Read)
  verification: Verify implementation correctness (Tools: All tools except Write/Edit)
```

### 5.3 监控代理状态

**查看后台任务**：

```
↓  # 按下箭头键查看任务列表
```

**任务状态类型**：

| 状态 | 说明 |
|------|------|
| `running` | 正在执行 |
| `completed` | 成功完成 |
| `failed` | 执行失败 |
| `killed` | 用户终止 |

**Agent进度追踪**（`tools/AgentTool/agentToolUtils.ts:369-387`）：

```typescript
function emitTaskProgress(
  tracker: ProgressTracker,
  taskId: string,
  toolUseId: string,
  description: string,
  startTime: number,
  lastToolName: string,
): void {
  const progress = getProgressUpdate(tracker)
  emitTaskProgressEvent({
    taskId,
    toolUseId,
    description: progress.lastActivity?.activityDescription ?? description,
    startTime,
    totalTokens: progress.tokenCount,
    toolUses: progress.toolUseCount,
    lastToolName,
  })
}
```

### 5.4 最佳实践

**提示词编写**（`tools/AgentTool/prompt.ts:99-113`）：

```markdown
## Writing the prompt

Brief the agent like a smart colleague who just walked into the room — it hasn't 
seen this conversation, doesn't know what you've tried, doesn't understand why 
this task matters.

- Explain what you're trying to accomplish and why.
- Describe what you've already learned or ruled out.
- Give enough context about the surrounding problem.
- If you need a short response, say so ("report in under 200 words").

**Never delegate understanding.** Don't write "based on your findings, fix the bug"
or "based on the research, implement it." Those phrases push synthesis onto the 
agent instead of doing it yourself.
```

**何时使用Agent**：

| 场景 | 推荐 |
|------|------|
| 大范围代码搜索 | ✅ Explore Agent |
| 实现方案规划 | ✅ Plan Agent |
| 功能验证测试 | ✅ Verification Agent |
| 读取单个文件 | ❌ 直接用Read工具 |
| 简单关键字搜索 | ❌ 直接用Grep工具 |
| 编辑特定文件 | ❌ 直接用Edit工具 |

**Fork使用场景**（`tools/AgentTool/prompt.ts:80-96`）：

```markdown
## When to fork

Fork yourself (omit `subagent_type`) when the intermediate tool output isn't 
worth keeping in your context. The criterion is qualitative — "will I need this 
output again" — not task size.

- **Research**: fork open-ended questions. If research can be broken into 
  independent questions, launch parallel forks in one message.
- **Implementation**: prefer to fork implementation work that requires more than 
  a couple of edits.

Forks are cheap because they share your prompt cache. Don't set `model` on a fork 
— a different model can't reuse the parent's cache.
```

---

## 6. 流式结果处理

### 6.1 task-notification格式

**通知事件结构**：

```typescript
// SDK事件
enqueueSdkEvent({
  type: 'system',
  subtype: 'task_notification',
  task_id: foregroundTaskId,
  tool_use_id: toolUseContext.toolUseId,
  status: 'completed' | 'failed' | 'stopped',
  output_file: '',
  summary: description,
  usage: {
    total_tokens: progress.tokenCount,
    tool_uses: progress.toolUseCount,
    duration_ms: Date.now() - agentStartTime
  }
})
```

**Agent通知**（`tools/AgentTool/agentToolUtils.ts:624-637`）：

```typescript
enqueueAgentNotification({
  taskId,
  description,
  status: 'completed',
  setAppState: rootSetAppState,
  finalMessage,
  usage: {
    totalTokens: getTokenCountFromTracker(tracker),
    toolUses: agentResult.totalToolUseCount,
    durationMs: agentResult.totalDurationMs,
  },
  toolUseId: toolUseContext.toolUseId,
  ...worktreeResult,
})
```

### 6.2 结果合并

**同步Agent结果处理**（`tools/AgentTool/agentToolUtils.ts:276-357`）：

```typescript
function finalizeAgentTool(
  agentMessages: MessageType[],
  agentId: string,
  metadata: {...},
): AgentToolResult {
  // 获取最后一条助手消息
  const lastAssistantMessage = getLastAssistantMessage(agentMessages)
  
  // 提取文本内容
  let content = lastAssistantMessage.message.content.filter(_ => _.type === 'text')
  
  // 如果最后消息没有文本，往前查找
  if (content.length === 0) {
    for (let i = agentMessages.length - 1; i >= 0; i--) {
      if (agentMessages[i].type !== 'assistant') continue
      const textBlocks = agentMessages[i].message.content.filter(_ => _.type === 'text')
      if (textBlocks.length > 0) {
        content = textBlocks
        break
      }
    }
  }
  
  // 计算统计数据
  const totalTokens = getTokenCountFromUsage(lastAssistantMessage.message.usage)
  const totalToolUseCount = countToolUses(agentMessages)
  
  // 记录完成事件
  logEvent('tengu_agent_tool_completed', {
    agent_type: agentType,
    model: resolvedAgentModel,
    total_tool_uses: totalToolUseCount,
    duration_ms: Date.now() - startTime,
    total_tokens: totalTokens,
    is_built_in_agent: isBuiltInAgent,
    is_async: isAsync,
  })
  
  return {
    agentId,
    agentType,
    content,
    totalDurationMs: Date.now() - startTime,
    totalTokens,
    totalToolUseCount,
    usage: lastAssistantMessage.message.usage,
  }
}
```

**进度消息处理**（`tools/AgentTool/UI.tsx:100-180`）：

```typescript
function processProgressMessages(
  messages: ProgressMessage<Progress>[],
  tools: Tools,
  isAgentRunning: boolean
): ProcessedMessage[] {
  const result: ProcessedMessage[] = []
  let currentGroup: {...} | null = null
  
  // 分组连续的搜索/读取操作
  for (const msg of agentMessages) {
    const info = getSearchOrReadInfo(msg, tools, toolUseByID)
    
    if (info && (info.isSearch || info.isRead || info.isREPL)) {
      // 累加到当前组
      if (!currentGroup) {
        currentGroup = { searchCount: 0, readCount: 0, replCount: 0, startUuid: msg.uuid }
      }
      // 计数...
    } else {
      // 非搜索/读取消息 - 输出当前组
      flushGroup(false)
      result.push({ type: 'original', message: msg })
    }
  }
  
  return result
}
```

---

## 7. 从零实现

### 7.1 核心文件结构

```
tools/AgentTool/
├── AgentTool.tsx          # 主工具定义和执行逻辑
├── UI.tsx                 # 用户界面渲染
├── agentToolUtils.ts      # 工具解析和结果处理
├── loadAgentsDir.ts       # Agent定义加载
├── prompt.ts              # 工具提示词生成
├── runAgent.ts            # Agent执行引擎
├── resumeAgent.ts         # Agent恢复逻辑
├── forkSubagent.ts        # Fork子代理实现
├── builtInAgents.ts       # 内置Agent注册
├── constants.ts           # 常量定义
├── agentColorManager.ts   # 颜色管理
├── agentDisplay.ts        # 显示工具函数
├── agentMemory.ts         # 持久化内存
├── agentMemorySnapshot.ts # 内存快照
└── built-in/              # 内置Agent定义
    ├── generalPurposeAgent.ts
    ├── exploreAgent.ts
    ├── planAgent.ts
    ├── verificationAgent.ts
    ├── claudeCodeGuideAgent.ts
    └── statuslineSetup.ts
```

### 7.2 最小实现示例

**Step 1: 定义Agent工具Schema**

```typescript
// tools/AgentTool/AgentTool.tsx
import { buildTool } from 'src/Tool.js'
import { z } from 'zod/v4'

const inputSchema = z.object({
  description: z.string().describe('A short (3-5 word) description of the task'),
  prompt: z.string().describe('The task for the agent to perform'),
  subagent_type: z.string().optional().describe('The type of specialized agent'),
  model: z.enum(['sonnet', 'opus', 'haiku']).optional(),
  run_in_background: z.boolean().optional(),
})

const outputSchema = z.union([
  z.object({
    status: z.literal('completed'),
    agentId: z.string(),
    content: z.array(z.object({ type: z.literal('text'), text: z.string() })),
    totalToolUseCount: z.number(),
    totalTokens: z.number(),
    totalDurationMs: z.number(),
  }),
  z.object({
    status: z.literal('async_launched'),
    agentId: z.string(),
    outputFile: z.string(),
  }),
])

export const AgentTool = buildTool({
  name: 'Agent',
  inputSchema,
  outputSchema,
  async call(input, context) {
    // 实现调用逻辑...
  },
})
```

**Step 2: 实现Agent执行函数**

```typescript
// tools/AgentTool/runAgent.ts
export async function* runAgent({
  agentDefinition,
  promptMessages,
  toolUseContext,
  isAsync,
}): AsyncGenerator<Message, void> {
  // 1. 创建Agent ID
  const agentId = createAgentId()
  
  // 2. 构建系统提示
  const systemPrompt = await getAgentSystemPrompt(agentDefinition, toolUseContext)
  
  // 3. 解析工具
  const resolvedTools = resolveAgentTools(agentDefinition, availableTools, isAsync)
  
  // 4. 创建子代理上下文
  const agentToolUseContext = createSubagentContext(toolUseContext, {
    options: { tools: resolvedTools, ... },
    agentId,
    agentType: agentDefinition.agentType,
    messages: promptMessages,
  })
  
  // 5. 执行查询循环
  try {
    for await (const message of query({
      messages: promptMessages,
      systemPrompt,
      toolUseContext: agentToolUseContext,
    })) {
      yield message
    }
  } finally {
    // 清理资源
    await cleanup(agentId)
  }
}
```

**Step 3: 实现Agent定义加载**

```typescript
// tools/AgentTool/loadAgentsDir.ts
export function parseAgentFromMarkdown(
  filePath: string,
  frontmatter: Record<string, unknown>,
  content: string,
  source: SettingSource,
): CustomAgentDefinition | null {
  const agentType = frontmatter['name']
  const whenToUse = frontmatter['description']
  
  if (!agentType || !whenToUse) return null
  
  return {
    agentType,
    whenToUse,
    tools: frontmatter['tools'],
    disallowedTools: frontmatter['disallowedTools'],
    model: frontmatter['model'],
    color: frontmatter['color'],
    getSystemPrompt: () => content.trim(),
    source,
  }
}
```

**Step 4: 实现异步Agent生命周期**

```typescript
// tools/AgentTool/agentToolUtils.ts
export async function runAsyncAgentLifecycle({
  taskId,
  abortController,
  makeStream,
  metadata,
}): Promise<void> {
  const agentMessages: MessageType[] = []
  
  try {
    const tracker = createProgressTracker()
    
    for await (const message of makeStream(onCacheSafeParams)) {
      agentMessages.push(message)
      updateProgressFromMessage(tracker, message)
      updateAsyncAgentProgress(taskId, getProgressUpdate(tracker))
    }
    
    const agentResult = finalizeAgentTool(agentMessages, taskId, metadata)
    completeAsyncAgent(agentResult, rootSetAppState)
    
    enqueueAgentNotification({
      taskId,
      status: 'completed',
      finalMessage: extractTextContent(agentResult.content),
    })
    
  } catch (error) {
    if (error instanceof AbortError) {
      killAsyncAgent(taskId, rootSetAppState)
      enqueueAgentNotification({ taskId, status: 'killed' })
    } else {
      failAsyncAgent(taskId, errorMessage(error), rootSetAppState)
      enqueueAgentNotification({ taskId, status: 'failed', error: errorMessage(error) })
    }
  } finally {
    clearInvokedSkillsForAgent(agentIdForCleanup)
    clearDumpState(agentIdForCleanup)
  }
}
```

### 7.3 关键设计决策

**1. 同步 vs 异步执行选择**

```typescript
// 同步：需要立即结果的场景
if (!shouldRunAsync) {
  // 在当前上下文中执行，共享abortController
  return runWithAgentContext(syncAgentContext, async () => {
    for await (const msg of runAgent({...})) {
      // 处理消息...
    }
    return finalizeAgentTool(agentMessages, agentId, metadata)
  })
}

// 异步：可并行或长时间任务
if (shouldRunAsync) {
  // 注册后台任务
  const task = registerAsyncAgent({...})
  
  // 后台执行
  void runAsyncAgentLifecycle({...})
  
  // 立即返回
  return { status: 'async_launched', agentId: task.agentId, ... }
}
```

**2. Fork机制的缓存优化**

```typescript
// Fork子代理继承父代理的完整上下文和系统提示
// 实现缓存共享，减少API调用

export function buildForkedMessages(
  directive: string,
  assistantMessage: AssistantMessage,
): MessageType[] {
  // 克隆完整的助手消息（所有tool_use块）
  const fullAssistantMessage = {
    ...assistantMessage,
    message: { ...assistantMessage.message, content: [...content] },
  }
  
  // 为所有tool_use构建占位符结果（缓存共享关键）
  const toolResultBlocks = toolUseBlocks.map(block => ({
    type: 'tool_result',
    tool_use_id: block.id,
    content: [{ type: 'text', text: FORK_PLACEHOLDER_RESULT }], // 统一占位符
  }))
  
  // 返回: [完整助手消息, 占位符结果 + 子任务指令]
  return [fullAssistantMessage, toolResultMessage]
}
```

**3. 工具过滤的两级机制**

```typescript
// 第一级：全局过滤（ALL_AGENT_DISALLOWED_TOOLS）
// 第二级：Agent自定义过滤（disallowedTools）
// 第三级：异步Agent特殊限制（ASYNC_AGENT_ALLOWED_TOOLS）

function filterToolsForAgent({ tools, isBuiltIn, isAsync }): Tools {
  return tools.filter(tool => {
    if (ALL_AGENT_DISALLOWED_TOOLS.has(tool.name)) return false
    if (!isBuiltIn && CUSTOM_AGENT_DISALLOWED_TOOLS.has(tool.name)) return false
    if (isAsync && !ASYNC_AGENT_ALLOWED_TOOLS.has(tool.name)) return false
    return true
  })
}
```

---

## 附录：关键文件路径

| 功能 | 文件路径 |
|------|----------|
| 主工具定义 | `tools/AgentTool/AgentTool.tsx` |
| UI渲染 | `tools/AgentTool/UI.tsx` |
| Agent执行引擎 | `tools/AgentTool/runAgent.ts` |
| Agent定义加载 | `tools/AgentTool/loadAgentsDir.ts` |
| 工具权限解析 | `tools/AgentTool/agentToolUtils.ts` |
| Fork子代理 | `tools/AgentTool/forkSubagent.ts` |
| Agent恢复 | `tools/AgentTool/resumeAgent.ts` |
| 提示词生成 | `tools/AgentTool/prompt.ts` |
| 内置Agent | `tools/AgentTool/built-in/*.ts` |
| 任务注册 | `tasks/LocalAgentTask/LocalAgentTask.ts` |

---

*分析完成于 2026-04-02*