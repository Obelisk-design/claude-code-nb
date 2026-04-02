# Plan规划模式详解

**分析日期:** 2026-04-02

## 目录

1. [Plan模式概述](#1-plan模式概述)
2. [commands/plan/ 命令目录](#2-commandsplan-命令目录)
3. [EnterPlanMode 工具](#3-enterplanmode-工具)
4. [ExitPlanMode 工具](#4-exitplanmode-工具)
5. [规划流程详解](#5-规划流程详解)
6. [用户使用指南](#6-用户使用指南)
7. [从零实现](#7-从零实现)

---

## 1. Plan模式概述

### 1.1 什么是Plan模式

Plan模式是Claude Code CLI的一种特殊权限模式，用于在执行复杂任务前进行**探索、设计和审批**。在该模式下：

- Claude **只能读取文件**，不能写入或编辑
- Claude 会**探索代码库**，理解现有架构和模式
- Claude 会**设计实现方案**并写入计划文件
- 用户**审批计划**后才能开始实施

### 1.2 Plan模式的核心价值

```
Plan模式的核心目的:
┌─────────────────────────────────────────────────────────────────┐
│  1. 防止方向错误 - 在写代码前确认用户意图                        │
│  2. 减少返工成本 - 复杂任务的架构决策需要用户确认                │
│  3. 提供实施蓝图 - 计划文件作为实施阶段的参考                    │
│  4. 支持团队协作 - Teammate可通过计划审批流程协同                │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 Plan模式与其他模式的关系

Plan模式是三种权限模式之一：

| 模式 | 说明 | 文件修改 | 权限要求 |
|------|------|----------|----------|
| `default` | 默认模式，每步需确认 | 允许 | 每次工具调用需用户批准 |
| `plan` | 规划模式，只读探索 | 禁止 | 只读操作自动允许 |
| `auto` | 自动模式，AI自主执行 | 允许 | AI自动判断权限 |

### 1.4 核心文件位置

```
Plan模式核心文件:
├── commands/plan/
│   ├── index.ts           # 命令入口定义
│   └── plan.tsx           # /plan 命令实现
├── tools/EnterPlanModeTool/
│   ├── EnterPlanModeTool.ts  # 进入Plan模式的工具定义
│   ├── prompt.ts            # 工具提示词（何时使用）
│   ├── constants.ts         # 工具名称常量
│   └── UI.tsx               # UI渲染组件
├── tools/ExitPlanModeTool/
│   ├── ExitPlanModeV2Tool.ts # 退出Plan模式的工具定义
│   ├── prompt.ts            # 工具提示词
│   ├── constants.ts         # 工具名称常量
│   └── UI.tsx               # UI渲染组件
├── utils/plans.ts          # 计划文件管理工具函数
├── utils/planModeV2.ts     # Plan模式V2配置
├── utils/permissions/
│   ├── permissionSetup.ts   # 权限模式切换逻辑
│   └── PermissionMode.ts    # 模式定义
├── components/permissions/
│   ├── EnterPlanModePermissionRequest/  # 进入Plan模式对话框
│   └── ExitPlanModePermissionRequest/   # 退出Plan模式对话框
├── bootstrap/state.ts      # 全局状态管理
└────────────────────────────
```

---

## 2. commands/plan/ 命令目录

### 2.1 命令入口 - `index.ts`

**文件路径:** `commands/plan/index.ts`

```typescript
import type { Command } from '../../commands.js'

const plan = {
  type: 'local-jsx',
  name: 'plan',
  description: 'Enable plan mode or view the current session plan',
  argumentHint: '[open|<description>]',
  load: () => import('./plan.js'),
} satisfies Command

export default plan
```

**逐行分析:**

| 行号 | 代码 | 说明 |
|------|------|------|
| 1 | `import type { Command }` | 导入命令类型定义 |
| 3-8 | `const plan = {...}` | 定义plan命令对象 |
| 4 | `type: 'local-jsx'` | 命令类型为本地JSX组件 |
| 5 | `name: 'plan'` | 命令名称，对应 `/plan` |
| 6 | `description: '...'` | 命令描述，帮助文本 |
| 7 | `argumentHint: '[open|<description>]'` | 参数提示：`open` 打开编辑，或描述 |
| 8 | `load: () => import('./plan.js')` | 懒加载实现模块 |

### 2.2 命令实现 - `plan.tsx`

**文件路径:** `commands/plan/plan.tsx`

#### 2.2.1 核心函数签名

```typescript
export async function call(
  onDone: LocalJSXCommandOnDone,
  context: LocalJSXCommandContext,
  args: string
): Promise<React.ReactNode>
```

#### 2.2.2 执行流程详解

**Step 1: 获取当前状态**

```typescript
const { getAppState, setAppState } = context
const appState = getAppState()
const currentMode = appState.toolPermissionContext.mode
```

- 从context获取状态读写函数
- 检查当前权限模式

**Step 2: 判断是否已在Plan模式**

```typescript
if (currentMode !== 'plan') {
  // 未在plan模式 -> 启用plan模式
  handlePlanModeTransition(currentMode, 'plan')
  setAppState(prev => ({
    ...prev,
    toolPermissionContext: applyPermissionUpdate(
      prepareContextForPlanMode(prev.toolPermissionContext),
      { type: 'setMode', mode: 'plan', destination: 'session' }
    ),
  }))
  // ...
}
```

**关键函数调用链:**

```
进入Plan模式:
┌──────────────────────────────────────────────────────────────┐
│  handlePlanModeTransition(currentMode, 'plan')               │
│  └─ 清除needsPlanModeExitAttachment标志                      │
│                                                              │
│  prepareContextForPlanMode(prev.toolPermissionContext)       │
│  └─ 保存prePlanMode（退出时恢复）                            │
│  └─ 处理auto模式的特殊情况                                   │
│                                                              │
│  applyPermissionUpdate(..., { type: 'setMode', mode: 'plan' })│
│  └ 更新权限上下文为plan模式                                  │
│                                                              │
│  setAppState(...)                                            │
│  └ 更新全局应用状态                                          │
└──────────────────────────────────────────────────────────────┘
```

**Step 3: 已在Plan模式时显示计划**

```typescript
// Already in plan mode - show the current plan
const planContent = getPlan()
const planPath = getPlanFilePath()
if (!planContent) {
  onDone('Already in plan mode. No plan written yet.')
  return null
}
```

**Step 4: `/plan open` 处理**

```typescript
const argList = args.trim().split(/\s+/)
if (argList[0] === 'open') {
  const result = await editFileInEditor(planPath)
  if (result.error) {
    onDone(`Failed to open plan in editor: ${result.error}`)
  } else {
    onDone(`Opened plan in editor: ${planPath}`)
  }
  return null
}
```

### 2.3 PlanDisplay UI组件

```tsx
function PlanDisplay({ planContent, planPath, editorName }) {
  return (
    <Box flexDirection="column">
      <Text bold>Current Plan</Text>
      <Text dimColor>{planPath}</Text>
      <Box marginTop={1}>
        <Text>{planContent}</Text>
      </Box>
      {editorName && (
        <Box marginTop={1}>
          <Text dimColor>"</plan open"</Text>
          <Text dimColor> to edit this plan in </Text>
          <Text bold dimColor>{editorName}</Text>
        </Box>
      )}
    </Box>
  )
}
```

---

## 3. EnterPlanMode 工具

### 3.1 工具定义 - `EnterPlanModeTool.ts`

**文件路径:** `tools/EnterPlanModeTool/EnterPlanModeTool.ts`

#### 3.1.1 Schema定义

```typescript
const inputSchema = lazySchema(() =>
  z.strictObject({
    // No parameters needed - 进入Plan模式无需参数
  }),
)

const outputSchema = lazySchema(() =>
  z.object({
    message: z.string().describe('Confirmation that plan mode was entered'),
  }),
)
```

#### 3.1.2 工具构建

```typescript
export const EnterPlanModeTool: Tool<InputSchema, Output> = buildTool({
  name: ENTER_PLAN_MODE_TOOL_NAME,  // 'EnterPlanMode'
  searchHint: 'switch to plan mode to design an approach before coding',
  maxResultSizeChars: 100_000,
  
  async description() {
    return 'Requests permission to enter plan mode for complex tasks'
  },
  
  async prompt() {
    return getEnterPlanModeToolPrompt()  // 动态提示词
  },
  
  shouldDefer: true,  // 需要用户确认
  
  isEnabled() {
    // --channels模式下禁用（无法显示审批对话框）
    if (feature('KAIROS_CHANNELS') && getAllowedChannels().length > 0) {
      return false
    }
    return true
  },
  
  isConcurrencySafe() { return true },
  isReadOnly() { return true },  // 只读操作
  
  // ... 其他方法
})
```

#### 3.1.3 call() 方法详解

```typescript
async call(_input, context) {
  // 1. 禁止在Agent上下文中使用
  if (context.agentId) {
    throw new Error('EnterPlanMode tool cannot be used in agent contexts')
  }

  // 2. 处理模式过渡
  const appState = context.getAppState()
  handlePlanModeTransition(appState.toolPermissionContext.mode, 'plan')

  // 3. 更新权限上下文
  context.setAppState(prev => ({
    ...prev,
    toolPermissionContext: applyPermissionUpdate(
      prepareContextForPlanMode(prev.toolPermissionContext),
      { type: 'setMode', mode: 'plan', destination: 'session' },
    ),
  }))

  // 4. 返回成功消息
  return {
    data: {
      message: 'Entered plan mode. You should now focus on exploring...',
    },
  }
}
```

#### 3.1.4 mapToolResultToToolResultBlockParam()

```typescript
mapToolResultToToolResultBlockParam({ message }, toolUseID) {
  const instructions = isPlanModeInterviewPhaseEnabled()
    ? `${message}

DO NOT write or edit any files except the plan file. 
Detailed workflow instructions will follow.`
    : `${message}

In plan mode, you should:
1. Thoroughly explore the codebase to understand existing patterns
2. Identify similar features and architectural approaches
3. Consider multiple approaches and their trade-offs
4. Use AskUserQuestion if you need to clarify the approach
5. Design a concrete implementation strategy
6. When ready, use ExitPlanMode to present your plan for approval

Remember: DO NOT write or edit any files yet. This is a read-only phase.`

  return {
    type: 'tool_result',
    content: instructions,
    tool_use_id: toolUseID,
  }
}
```

### 3.2 提示词策略 - `prompt.ts`

**文件路径:** `tools/EnterPlanModeTool/prompt.ts`

#### 3.2.1 外部用户版本

```typescript
function getEnterPlanModeToolPromptExternal(): string {
  return `Use this tool proactively when you're about to start a non-trivial 
implementation task...

## When to Use This Tool

**Prefer using EnterPlanMode** for implementation tasks unless they're simple:

1. **New Feature Implementation**: Adding meaningful new functionality
2. **Multiple Valid Approaches**: The task can be solved in several ways
3. **Code Modifications**: Changes that affect existing behavior
4. **Architectural Decisions**: Choosing between patterns or technologies
5. **Multi-File Changes**: Will likely touch more than 2-3 files
6. **Unclear Requirements**: Need to explore before understanding scope
7. **User Preferences Matter**: Implementation could go multiple ways

## When NOT to Use This Tool

- Single-line or few-line fixes (typos, obvious bugs)
- Adding a single function with clear requirements
- Tasks where user gave very specific, detailed instructions
- Pure research/exploration tasks (use Agent tool instead)
`
}
```

#### 3.2.2 Ant内部版本

```typescript
function getEnterPlanModeToolPromptAnt(): string {
  return `Use this tool when a task has genuine ambiguity about the right 
approach and getting user input before coding would prevent significant rework.

## When to Use This Tool

1. **Significant Architectural Ambiguity**: Multiple reasonable approaches
2. **Unclear Requirements**: Need to explore and clarify before progress
3. **High-Impact Restructuring**: Major restructuring, getting buy-in reduces risk

## When NOT to Use This Tool

- Task is straightforward even if it touches multiple files
- User's request is specific enough that implementation path is clear
- Adding feature with obvious implementation pattern
- Bug fixes where fix is clear once understood
- Research/exploration tasks (use Agent tool instead)
- User says "can we work on X" or "let's do X" — just get started
`
}
```

**关键差异:**

| 特性 | External版本 | Ant版本 |
|------|-------------|---------|
| 触发门槛 | 更保守（更多场景使用） | 更激进（减少使用） |
| 用户意图 | "can we work on X"仍建议规划 | 直接开始工作 |
| 理念 | 宁可规划也不要返工 | 宁可开始也不要过度规划 |

### 3.3 UI渲染 - `UI.tsx`

**文件路径:** `tools/EnterPlanModeTool/UI.tsx`

```tsx
export function renderToolResultMessage(output, ...): React.ReactNode {
  return (
    <Box flexDirection="column" marginTop={1}>
      <Box flexDirection="row">
        <Text color={getModeColor('plan')}>{BLACK_CIRCLE}</Text>
        <Text> Entered plan mode</Text>
      </Box>
      <Box paddingLeft={2}>
        <Text dimColor>
          Claude is now exploring and designing an implementation approach.
        </Text>
      </Box>
    </Box>
  )
}

export function renderToolUseRejectedMessage(): React.ReactNode {
  return (
    <Box flexDirection="row" marginTop={1}>
      <Text color={getModeColor('default')}>{BLACK_CIRCLE}</Text>
      <Text> User declined to enter plan mode</Text>
    </Box>
  )
}
```

---

## 4. ExitPlanMode 工具

### 4.1 工具定义 - `ExitPlanModeV2Tool.ts`

**文件路径:** `tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts`

#### 4.1.1 Schema定义

```typescript
// 输入Schema
const allowedPromptSchema = lazySchema(() =>
  z.object({
    tool: z.enum(['Bash']),
    prompt: z.string().describe('Semantic description of the action'),
  }),
)

const inputSchema = lazySchema(() =>
  z.strictObject({
    allowedPrompts: z.array(allowedPromptSchema()).optional(),
  }).passthrough(),
)

// 输出Schema
const outputSchema = lazySchema(() =>
  z.object({
    plan: z.string().nullable(),
    isAgent: z.boolean(),
    filePath: z.string().optional(),
    hasTaskTool: z.boolean().optional(),
    planWasEdited: z.boolean().optional(),
    awaitingLeaderApproval: z.boolean().optional(),
    requestId: z.string().optional(),
  }),
)
```

#### 4.1.2 工具构建核心配置

```typescript
export const ExitPlanModeV2Tool: Tool<InputSchema, Output> = buildTool({
  name: EXIT_PLAN_MODE_V2_TOOL_NAME,  // 'ExitPlanMode'
  searchHint: 'present plan for approval and start coding (plan mode only)',
  
  shouldDefer: true,  // 需要用户确认
  
  requiresUserInteraction() {
    // Teammate不需要本地用户交互
    if (isTeammate()) return false
    // 非Teammate需要用户确认
    return true
  },
  
  async validateInput(_input, { getAppState, options }) {
    // 必须在plan模式下调用
    const mode = getAppState().toolPermissionContext.mode
    if (mode !== 'plan') {
      logEvent('tengu_exit_plan_mode_called_outside_plan', {...})
      return {
        result: false,
        message: 'You are not in plan mode...',
      }
    }
    return { result: true }
  },
  
  async checkPermissions(input, context) {
    if (isTeammate()) {
      return { behavior: 'allow', updatedInput: input }
    }
    return { behavior: 'ask', message: 'Exit plan mode?', updatedInput: input }
  },
})
```

#### 4.1.3 call() 方法完整流程

```typescript
async call(input, context) {
  const isAgent = !!context.agentId
  const filePath = getPlanFilePath(context.agentId)
  
  // 1. 获取计划内容（优先使用编辑后的版本）
  const inputPlan = 'plan' in input && typeof input.plan === 'string' 
    ? input.plan : undefined
  const plan = inputPlan ?? getPlan(context.agentId)
  
  // 2. 同步磁盘（用户可能编辑了计划）
  if (inputPlan !== undefined && filePath) {
    await writeFile(filePath, inputPlan, 'utf-8')
    void persistFileSnapshotIfRemote()
  }
  
  // 3. Teammate审批流程
  if (isTeammate() && isPlanModeRequired()) {
    // 发送审批请求到team-lead
    const approvalRequest = {
      type: 'plan_approval_request',
      from: agentName,
      planFilePath: filePath,
      planContent: plan,
      requestId,
    }
    await writeToMailbox('team-lead', approvalRequest, teamName)
    
    return {
      data: {
        plan,
        isAgent: true,
        awaitingLeaderApproval: true,
        requestId,
      },
    }
  }
  
  // 4. 本地用户审批流程
  context.setAppState(prev => {
    if (prev.toolPermissionContext.mode !== 'plan') return prev
    
    setHasExitedPlanMode(true)
    setNeedsPlanModeExitAttachment(true)
    
    // 恢复之前的模式
    let restoreMode = prev.toolPermissionContext.prePlanMode ?? 'default'
    
    // 处理auto模式门控
    if (restoreMode === 'auto' && !isAutoModeGateEnabled()) {
      restoreMode = 'default'  // 门控关闭时回退到default
    }
    
    // 恢复危险权限（如果在auto模式下被剥离）
    let baseContext = prev.toolPermissionContext
    if (!restoringToAuto && prev.toolPermissionContext.strippedDangerousRules) {
      baseContext = restoreDangerousPermissions(baseContext)
    }
    
    return {
      ...prev,
      toolPermissionContext: {
        ...baseContext,
        mode: restoreMode,
        prePlanMode: undefined,
      },
    }
  })
  
  return {
    data: { plan, isAgent, filePath, planWasEdited: inputPlan !== undefined },
  }
}
```

### 4.2 提示词 - `prompt.ts`

**文件路径:** `tools/ExitPlanModeTool/prompt.ts`

```typescript
export const EXIT_PLAN_MODE_V2_TOOL_PROMPT = `Use this tool when you are in 
plan mode and have finished writing your plan to the plan file and are ready 
for user approval.

## How This Tool Works
- You should have already written your plan to the plan file
- This tool does NOT take the plan content as a parameter
- It reads the plan from the file you wrote
- This tool simply signals that you're done planning

## When to Use This Tool
IMPORTANT: Only use this tool when the task requires planning the implementation 
steps of a task that requires writing code. For research tasks where you're 
gathering information - do NOT use this tool.

## Before Using This Tool
Ensure your plan is complete and unambiguous:
- If you have unresolved questions, use AskUserQuestion first
- Once your plan is finalized, use THIS tool to request approval

**Important:** Do NOT use AskUserQuestion to ask "Is this plan okay?" - 
that's exactly what THIS tool does.
`
```

### 4.3 mapToolResultToToolResultBlockParam()

```typescript
mapToolResultToToolResultBlockParam(output, toolUseID) {
  // 1. Teammate等待审批
  if (output.awaitingLeaderApproval) {
    return {
      type: 'tool_result',
      content: `Your plan has been submitted to the team lead for approval.

Plan file: ${output.filePath}

**What happens next:**
1. Wait for the team lead to review your plan
2. You will receive a message in your inbox
3. If approved, you can proceed with implementation

Request ID: ${output.requestId}`,
      tool_use_id: toolUseID,
    }
  }
  
  // 2. Agent子代理
  if (output.isAgent) {
    return {
      type: 'tool_result',
      content: 'User has approved the plan. Please respond with "ok"',
      tool_use_id: toolUseID,
    }
  }
  
  // 3. 空计划
  if (!output.plan || output.plan.trim() === '') {
    return {
      type: 'tool_result',
      content: 'User has approved exiting plan mode. You can now proceed.',
      tool_use_id: toolUseID,
    }
  }
  
  // 4. 正常审批
  const planLabel = output.planWasEdited 
    ? 'Approved Plan (edited by user)' 
    : 'Approved Plan'
  
  return {
    type: 'tool_result',
    content: `User has approved your plan. You can now start coding.

Your plan has been saved to: ${output.filePath}

## ${planLabel}:
${output.plan}`,
    tool_use_id: toolUseID,
  }
}
```

---

## 5. 规划流程详解

### 5.1 完整流程图

```
Plan模式完整流程:
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  用户发起请求                                                            │
│       │                                                                 │
│       ▼                                                                 │
│  ┌──────────────────┐                                                   │
│  │ Claude判断是否需要│                                                   │
│  │ 进入Plan模式      │                                                   │
│  └──────────────────┘                                                   │
│       │                                                                 │
│       ├── 简单任务 ────────────────────────────────▶ 直接执行           │
│       │                                                                 │
│       ▼ 复杂任务                                                        │
│  ┌──────────────────┐                                                   │
│  │ 调用EnterPlanMode │                                                   │
│  │ Tool              │                                                   │
│  └──────────────────┘                                                   │
│       │                                                                 │
│       ▼                                                                 │
│  ┌──────────────────────────────────────────┐                           │
│  │ handlePlanModeTransition(from, 'plan')   │                           │
│  │ ─ 清除needsPlanModeExitAttachment        │                           │
│  └──────────────────────────────────────────┘                           │
│       │                                                                 │
│       ▼                                                                 │
│  ┌──────────────────────────────────────────┐                           │
│  │ prepareContextForPlanMode(context)       │                           │
│  │ ─ 保存prePlanMode                        │                           │
│  │ ─ 处理auto模式兼容                       │                           │
│  └──────────────────────────────────────────┘                           │
│       │                                                                 │
│       ▼                                                                 │
│  ┌──────────────────┐                                                   │
│  │ 用户审批对话框    │◀── EnterPlanModePermissionRequest                 │
│  │ Yes / No         │                                                   │
│  └──────────────────┘                                                   │
│       │                                                                 │
│       ├── No ───────────────────────────▶ 保持原模式，直接执行          │
│       │                                                                 │
│       ▼ Yes                                                             │
│  ┌──────────────────────────────────────────────────────┐               │
│  │                    PLAN MODE                         │               │
│  │                                                      │               │
│  │  ┌────────────────────────────────────────────────┐ │               │
│  │  │ Phase 1: 探索代码库                            │ │               │
│  │  │ ─ 使用Glob/Grep/Read工具                       │ │               │
│  │  │ ─ 理解现有架构和模式                           │ │               │
│  │  └────────────────────────────────────────────────┘ │               │
│  │                      │                              │               │
│  │                      ▼                              │               │
│  │  ┌────────────────────────────────────────────────┐ │               │
│  │  │ Phase 2: 设计实施方案                          │ │               │
│  │  │ ─ 考虑多种方案及权衡                           │ │               │
│  │  │ ─ 使用AskUserQuestion澄清需求                  │ │               │
│  │  └────────────────────────────────────────────────┘ │               │
│  │                      │                              │               │
│  │                      ▼                              │               │
│  │  ┌────────────────────────────────────────────────┐ │               │
│  │  │ Phase 3: 撰写计划文件                          │ │               │
│  │  │ ─ 写入 {plansDir}/{slug}.md                    │ │               │
│  │  │ ─ 使用Write工具（唯一允许的写操作）            │ │               │
│  │  └────────────────────────────────────────────────┘ │               │
│  │                      │                              │               │
│  │                      ▼                              │               │
│  │  ┌────────────────────────────────────────────────┐ │               │
│  │  │ Phase 4: 调用ExitPlanMode                      │ │               │
│  │  └────────────────────────────────────────────────┘ │               │
│  │                                                      │               │
│  └──────────────────────────────────────────────────────┘               │
│       │                                                                 │
│       ▼                                                                 │
│  ┌──────────────────────────────────────────────────────┐               │
│  │ ExitPlanModePermissionRequest                        │               │
│  │ ─ 显示计划内容                                        │               │
│  │ ─ 用户可编辑计划                                      │               │
│  │ ─ 提供反馈                                            │               │
│  └──────────────────────────────────────────────────────┘               │
│       │                                                                 │
│       ├── Approve ──────────────────────────────────────────────┐       │
│       │                                                         │       │
│       │   ┌────────────────────────────────────────────────────┐│       │
│       │   │ restoreMode = prePlanMode ?? 'default'             ││       │
│       │   │ ─ 恢复之前的权限模式                               ││       │
│       │   │ ─ 恢复被剥离的危险权限                             ││       │
│       │   │ ─ setHasExitedPlanMode(true)                      ││       │
│       │   └────────────────────────────────────────────────────┘│       │
│       │                                                         │       │
│       │   ▼                                                     │       │
│       │   ┌────────────────────────────────────────────────────┐│       │
│       │   │              IMPLEMENTATION PHASE                  ││       │
│       │   │ ─ Claude按照计划执行                               ││       │
│       │   │ ─ 可以读写文件                                     ││       │
│       │   └────────────────────────────────────────────────────┘│       │
│       │                                                         │       │
│       ├── Reject ─────────────────────────────▶ 返回Plan模式    │       │
│       │                                                         │       │
│       └── Ultraplan ───────────────────────────▶ 启动多Agent    │       │
│                                                                 │       │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 状态转换详解

#### 5.2.1 handlePlanModeTransition()

**文件路径:** `bootstrap/state.ts:1349-1363`

```typescript
export function handlePlanModeTransition(
  fromMode: string,
  toMode: string,
): void {
  // 切换到plan模式时清除退出附件标志
  if (toMode === 'plan' && fromMode !== 'plan') {
    STATE.needsPlanModeExitAttachment = false
  }

  // 从plan模式切换出去时触发退出附件
  if (fromMode === 'plan' && toMode !== 'plan') {
    STATE.needsPlanModeExitAttachment = true
  }
}
```

**作用:** 管理 `needsPlanModeExitAttachment` 标志，防止快速切换时发送重复消息。

#### 5.2.2 prepareContextForPlanMode()

**文件路径:** `utils/permissions/permissionSetup.ts:1462-1493`

```typescript
export function prepareContextForPlanMode(
  context: ToolPermissionContext,
): ToolPermissionContext {
  const currentMode = context.mode
  if (currentMode === 'plan') return context
  
  if (feature('TRANSCRIPT_CLASSIFIER')) {
    const planAutoMode = shouldPlanUseAutoMode()
    
    // 从auto进入plan
    if (currentMode === 'auto') {
      if (planAutoMode) {
        // 保持auto激活，只保存prePlanMode
        return { ...context, prePlanMode: 'auto' }
      }
      // 否则剥离危险权限
      autoModeStateModule?.setAutoModeActive(false)
      return {
        ...restoreDangerousPermissions(context),
        prePlanMode: 'auto',
      }
    }
    
    // 从其他模式进入，激活auto（如果用户选择）
    if (planAutoMode && currentMode !== 'bypassPermissions') {
      autoModeStateModule?.setAutoModeActive(true)
      return {
        ...stripDangerousPermissionsForAutoMode(context),
        prePlanMode: currentMode,
      }
    }
  }
  
  // 纯plan模式进入
  return { ...context, prePlanMode: currentMode }
}
```

**核心逻辑:**

| 场景 | 处理方式 |
|------|----------|
| 从auto进入，planAutoMode=true | 保持auto激活，记录prePlanMode='auto' |
| 从auto进入，planAutoMode=false | 关闭auto，恢复危险权限 |
| 从default进入，planAutoMode=true | 激活auto，剥离危险权限 |
| 从default进入，planAutoMode=false | 纯plan模式 |
| 从bypassPermissions进入 | 不激活auto（安全考虑） |

### 5.3 计划文件管理 - `utils/plans.ts`

#### 5.3.1 计划文件路径生成

```typescript
export function getPlanFilePath(agentId?: AgentId): string {
  const planSlug = getPlanSlug(getSessionId())
  
  // 主会话
  if (!agentId) {
    return join(getPlansDirectory(), `${planSlug}.md`)
  }
  
  // 子Agent
  return join(getPlansDirectory(), `${planSlug}-agent-${agentId}.md`)
}
```

**命名规则:**
- 主会话: `{word-slug}.md`
- 子Agent: `{word-slug}-agent-{agentId}.md`

#### 5.3.2 计划目录位置

```typescript
export const getPlansDirectory = memoize(function getPlansDirectory(): string {
  const settings = getInitialSettings()
  const settingsDir = settings.plansDirectory
  
  if (settingsDir) {
    // 用户自定义目录（需在项目根目录内）
    const resolved = resolve(cwd, settingsDir)
    // 路径穿越检查
    if (!resolved.startsWith(cwd + sep) && resolved !== cwd) {
      plansPath = join(getClaudeConfigHomeDir(), 'plans')
    } else {
      plansPath = resolved
    }
  } else {
    // 默认目录
    plansPath = join(getClaudeConfigHomeDir(), 'plans')
  }
  
  // 确保目录存在
  getFsImplementation().mkdirSync(plansPath)
  
  return plansPath
})
```

**默认位置:** `~/.claude/plans/` 或用户自定义

#### 5.3.3 Slug生成机制

```typescript
export function getPlanSlug(sessionId?: SessionId): string {
  const id = sessionId ?? getSessionId()
  const cache = getPlanSlugCache()
  let slug = cache.get(id)
  
  if (!slug) {
    const plansDir = getPlansDirectory()
    // 尝试生成不冲突的slug（最多10次）
    for (let i = 0; i < MAX_SLUG_RETRIES; i++) {
      slug = generateWordSlug()  // 如 "quick-brown-fox"
      const filePath = join(plansDir, `${slug}.md`)
      if (!getFsImplementation().existsSync(filePath)) {
        break
      }
    }
    cache.set(id, slug!)
  }
  return slug!
}
```

---

## 6. 用户使用指南

### 6.1 手动触发Plan模式

**方式一：使用 `/plan` 命令**

```
用户输入: /plan
效果: 
  - 如果未在plan模式，进入plan模式
  - 如果已在plan模式，显示当前计划内容
```

**方式二：使用 `/plan open`**

```
用户输入: /plan open
效果: 在外部编辑器中打开计划文件
```

**方式三：使用 `/plan <描述>`**

```
用户输入: /plan add authentication feature
效果: 进入plan模式并携带描述（触发shouldQuery立即查询）
```

### 6.2 Plan模式下的用户界面

#### 6.2.1 EnterPlanMode审批对话框

```
┌───────────────────────────────────────────────────────┐
│ ○ Enter plan mode?                                    │
│                                                       │
│ Claude wants to enter plan mode to explore and        │
│ design an implementation approach.                    │
│                                                       │
│ In plan mode, Claude will:                            │
│  · Explore the codebase thoroughly                    │
│  · Identify existing patterns                         │
│  · Design an implementation strategy                  │
│  · Present a plan for your approval                   │
│                                                       │
│ No code changes will be made until you approve.       │
│                                                       │
│ ○ Yes, enter plan mode                                │
│ ○ No, start implementing now                          │
└───────────────────────────────────────────────────────┘
```

#### 6.2.2 ExitPlanMode审批对话框

```
┌───────────────────────────────────────────────────────┐
│ ○ User approved Claude's plan                         │
│                                                       │
│ Plan saved to: ~/.claude/plans/quick-fox.md · /plan   │
│                                                       │
│ ## Implementation Plan                                │
│                                                       │
│ 1. Create authentication middleware                   │
│ 2. Add login/logout routes                            │
│ 3. Implement session management                       │
│ 4. Add password hashing                               │
│                                                       │
│ ┌─────────────────────────────────────────────────┐   │
│ │ Options:                                         │   │
│ │ ○ Yes, proceed                                   │   │
│ │ ○ Edit plan in editor                            │   │
│ │ ○ No, revise                                     │   │
│ │ ○ Send to Ultraplan                              │   │
│ └─────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────┘
```

### 6.3 自动触发场景

Claude会自动判断并触发Plan模式：

```
自动触发EnterPlanMode的场景:
┌─────────────────────────────────────────────────────────┐
│ 用户请求                  │ Claude判断                   │
├─────────────────────────────────────────────────────────┤
│ "Add user authentication" │ 架构决策需要确认 → Plan模式   │
│ "Optimize database"       │ 多种方案可选 → Plan模式       │
│ "Refactor this module"    │ 影响范围大 → Plan模式         │
│ "Fix typo in README"      │ 简单修改 → 直接执行           │
│ "Add console.log"         │ 明确需求 → 直接执行           │
└─────────────────────────────────────────────────────────┘
```

### 6.4 Plan模式V2特性

**文件路径:** `utils/planModeV2.ts`

#### 6.4.1 Interview Phase

```typescript
export function isPlanModeInterviewPhaseEnabled(): boolean {
  // Ant用户始终启用
  if (process.env.USER_TYPE === 'ant') return true
  
  // 环境变量控制
  const env = process.env.CLAUDE_CODE_PLAN_MODE_INTERVIEW_PHASE
  if (isEnvTruthy(env)) return true
  if (isEnvDefinedFalsy(env)) return false
  
  // Feature gate控制
  return getFeatureValue_CACHED_MAY_BE_STALE(
    'tengu_plan_mode_interview_phase',
    false,
  )
}
```

**Interview Phase启用时的变化:**

| 特性 | 标准模式 | Interview Phase |
|------|----------|-----------------|
| 工作流指导 | tool_result中 | 系统消息附件 |
| 计划文件大小 | 无限制 | PewterLedger实验控制 |
| Agent数量 | 1 | 根据订阅等级分配 |

#### 6.4.2 Agent数量分配

```typescript
export function getPlanModeV2AgentCount(): number {
  // 环境变量优先
  if (process.env.CLAUDE_CODE_PLAN_V2_AGENT_COUNT) {
    const count = parseInt(process.env.CLAUDE_CODE_PLAN_V2_AGENT_COUNT, 10)
    if (!isNaN(count) && count > 0 && count <= 10) return count
  }
  
  // 订阅等级决定
  const subscriptionType = getSubscriptionType()
  const rateLimitTier = getRateLimitTier()
  
  if (subscriptionType === 'max' && rateLimitTier === 'default_claude_max_20x') {
    return 3
  }
  
  if (subscriptionType === 'enterprise' || subscriptionType === 'team') {
    return 3
  }
  
  return 1  // 默认单个Agent
}
```

### 6.5 Teammate计划审批

**文件路径:** `components/messages/PlanApprovalMessage.tsx`

当Teammate需要审批计划时：

```
┌───────────────────────────────────────────────────────┐
│ ○ Plan Approval Request from agent-1                  │
│                                                       │
│ ## Implementation Plan                                │
│ [计划内容...]                                          │
│                                                       │
│ Plan file: ~/.claude/plans/quick-fox-agent-agent1.md  │
└───────────────────────────────────────────────────────┘
```

审批响应：

```
┌───────────────────────────────────────────────────────┐
│ ✓ Plan Approved by team-lead                          │
│                                                       │
│ You can now proceed with implementation.              │
│ Your plan mode restrictions have been lifted.         │
└───────────────────────────────────────────────────────┘

或

┌───────────────────────────────────────────────────────┐
│ ✗ Plan Rejected by team-lead                          │
│                                                       │
│ Feedback: Need to add error handling section          │
│                                                       │
│ Please revise your plan based on the feedback.        │
└───────────────────────────────────────────────────────┘
```

---

## 7. 从零实现

### 7.1 最小化实现方案

以下是实现Plan模式的最小化代码框架：

#### 7.1.1 权限模式定义

```typescript
// types/PermissionMode.ts
export type PermissionMode = 'default' | 'plan' | 'auto' | 'bypassPermissions'

export function getModeColor(mode: PermissionMode): string {
  switch (mode) {
    case 'plan': return 'yellow'
    case 'auto': return 'cyan'
    case 'bypassPermissions': return 'red'
    default: return 'white'
  }
}
```

#### 7.1.2 EnterPlanMode工具

```typescript
// tools/EnterPlanModeTool.ts
import { z } from 'zod'
import { buildTool } from '../Tool.js'

const inputSchema = z.strictObject({})
const outputSchema = z.object({
  message: z.string(),
})

export const EnterPlanModeTool = buildTool({
  name: 'EnterPlanMode',
  
  async description() {
    return 'Requests permission to enter plan mode'
  },
  
  async prompt() {
    return `Use this tool for complex tasks requiring exploration before coding.
    
    When to use:
    - New feature implementation
    - Multiple valid approaches
    - Multi-file changes
    - Architectural decisions
    
    When NOT to use:
    - Simple fixes
    - Clear requirements
    - Research tasks`
  },
  
  shouldDefer: true,  // 需要用户确认
  isReadOnly: true,
  
  async call(_input, context) {
    // 保存当前模式
    const prePlanMode = context.getAppState().toolPermissionContext.mode
    
    // 切换到plan模式
    context.setAppState(prev => ({
      ...prev,
      toolPermissionContext: {
        ...prev.toolPermissionContext,
        mode: 'plan',
        prePlanMode,  // 保存以便退出时恢复
      },
    }))
    
    return {
      data: {
        message: 'Entered plan mode. Explore the codebase and design an approach.',
      },
    }
  },
  
  mapToolResultToToolResultBlockParam({ message }, toolUseID) {
    return {
      type: 'tool_result',
      content: `${message}\n\nIn plan mode:\n1. Explore the codebase\n2. Design implementation\n3. Write plan file\n4. Call ExitPlanMode\n\nDO NOT write any files yet.`,
      tool_use_id: toolUseID,
    }
  },
})
```

#### 7.1.3 ExitPlanMode工具

```typescript
// tools/ExitPlanModeTool.ts
import { z } from 'zod'
import { buildTool } from '../Tool.js'
import { getPlan, getPlanFilePath } from '../utils/plans.js'

const inputSchema = z.strictObject({})
const outputSchema = z.object({
  plan: z.string().nullable(),
  filePath: z.string(),
})

export const ExitPlanModeTool = buildTool({
  name: 'ExitPlanMode',
  
  async description() {
    return 'Signals that planning is complete, requests user approval'
  },
  
  async prompt() {
    return `Use this tool when you have finished writing your plan.
    
    Requirements:
    - You must be in plan mode
    - You must have written a plan file
    - The plan should be complete and unambiguous`
  },
  
  shouldDefer: true,
  
  async validateInput(_input, context) {
    const mode = context.getAppState().toolPermissionContext.mode
    if (mode !== 'plan') {
      return {
        result: false,
        message: 'Not in plan mode. Cannot exit.',
      }
    }
    return { result: true }
  },
  
  async checkPermissions(input, context) {
    return {
      behavior: 'ask',
      message: 'Exit plan mode?',
      updatedInput: input,
    }
  },
  
  async call(_input, context) {
    const filePath = getPlanFilePath()
    const plan = getPlan()
    
    // 恢复之前的模式
    const prePlanMode = context.getAppState().toolPermissionContext.prePlanMode ?? 'default'
    
    context.setAppState(prev => ({
      ...prev,
      toolPermissionContext: {
        ...prev.toolPermissionContext,
        mode: prePlanMode,
        prePlanMode: undefined,
      },
    }))
    
    return {
      data: { plan, filePath },
    }
  },
  
  mapToolResultToToolResultBlockParam({ plan, filePath }, toolUseID) {
    if (!plan) {
      return {
        type: 'tool_result',
        content: 'User approved. You can now proceed.',
        tool_use_id: toolUseID,
      }
    }
    
    return {
      type: 'tool_result',
      content: `User approved your plan.\n\nPlan saved to: ${filePath}\n\n## Approved Plan:\n${plan}`,
      tool_use_id: toolUseID,
    }
  },
})
```

#### 7.1.4 计划文件管理

```typescript
// utils/plans.ts
import { join } from 'path'
import { randomUUID } from 'crypto'
import { readFile, writeFile, mkdirSync } from 'fs'

const PLANS_DIR = join(process.env.HOME, '.claude', 'plans')

// 确保目录存在
mkdirSync(PLANS_DIR, { recursive: true })

// 会话slug缓存
const slugCache = new Map<string, string>()

export function generateSlug(): string {
  // 生成随机单词组合
  const words = ['quick', 'brown', 'fox', 'lazy', 'dog', 'bright', 'star']
  return `${words[Math.floor(Math.random() * words.length)]}-${randomUUID().slice(0, 8)}`
}

export function getPlanSlug(sessionId: string): string {
  if (!slugCache.has(sessionId)) {
    slugCache.set(sessionId, generateSlug())
  }
  return slugCache.get(sessionId)!
}

export function getPlanFilePath(sessionId: string): string {
  const slug = getPlanSlug(sessionId)
  return join(PLANS_DIR, `${slug}.md`)
}

export function getPlan(sessionId: string): string | null {
  const filePath = getPlanFilePath(sessionId)
  try {
    return readFile(filePath, 'utf-8')
  } catch {
    return null
  }
}

export function savePlan(sessionId: string, content: string): void {
  const filePath = getPlanFilePath(sessionId)
  writeFile(filePath, content, 'utf-8')
}
```

#### 7.1.5 /plan命令

```typescript
// commands/plan/index.ts
export const planCommand = {
  name: 'plan',
  description: 'Enable plan mode or view current plan',
  argumentHint: '[open]',
  
  async call(context, args) {
    const mode = context.getAppState().toolPermissionContext.mode
    
    // 未在plan模式 -> 启用
    if (mode !== 'plan') {
      context.setAppState(prev => ({
        ...prev,
        toolPermissionContext: {
          ...prev.toolPermissionContext,
          mode: 'plan',
          prePlanMode: mode,
        },
      }))
      return 'Enabled plan mode'
    }
    
    // 已在plan模式 -> 显示计划
    const plan = getPlan(context.sessionId)
    if (!plan) {
      return 'Already in plan mode. No plan written yet.'
    }
    
    // /plan open -> 打开编辑器
    if (args === 'open') {
      const filePath = getPlanFilePath(context.sessionId)
      // 打开外部编辑器
      await openInEditor(filePath)
      return `Opened plan in editor: ${filePath}`
    }
    
    // 显示计划内容
    return `Current Plan:\n\n${plan}`
  },
}
```

### 7.2 完整实现的额外考虑

#### 7.2.1 Auto模式兼容

```typescript
// permissionSetup.ts
export function prepareContextForPlanMode(context): ToolPermissionContext {
  const currentMode = context.mode
  const useAutoDuringPlan = shouldPlanUseAutoMode()
  
  // 从auto进入
  if (currentMode === 'auto') {
    if (useAutoDuringPlan) {
      // 保持auto激活
      return { ...context, prePlanMode: 'auto' }
    } else {
      // 关闭auto
      setAutoModeActive(false)
      return { ...restoreDangerousPermissions(context), prePlanMode: 'auto' }
    }
  }
  
  // 从其他模式进入，可选激活auto
  if (useAutoDuringPlan && currentMode !== 'bypassPermissions') {
    setAutoModeActive(true)
    return {
      ...stripDangerousPermissionsForAutoMode(context),
      prePlanMode: currentMode,
    }
  }
  
  return { ...context, prePlanMode: currentMode }
}
```

#### 7.2.2 Teammate支持

```typescript
// ExitPlanModeV2Tool.ts call()中的Teammate处理
if (isTeammate() && isPlanModeRequired()) {
  // 发送审批请求
  const approvalRequest = {
    type: 'plan_approval_request',
    from: getAgentName(),
    planFilePath: filePath,
    planContent: plan,
    requestId: generateRequestId(),
  }
  
  await writeToMailbox('team-lead', approvalRequest)
  
  return {
    data: {
      awaitingLeaderApproval: true,
      requestId,
    },
  }
}
```

#### 7.2.3 会话恢复

```typescript
// plans.ts
export async function copyPlanForResume(log, sessionId): Promise<boolean> {
  const slug = extractSlugFromLog(log)
  if (!slug) return false
  
  setPlanSlug(sessionId, slug)
  
  // 尝试读取现有计划文件
  const planPath = getPlanFilePath(sessionId)
  try {
    await readFile(planPath)
    return true
  } catch {
    // 从消息历史恢复
    const recovered = recoverPlanFromMessages(log)
    if (recovered) {
      await writeFile(planPath, recovered)
      return true
    }
    return false
  }
}
```

---

## 附录

### A. 关键函数速查表

| 函数 | 文件 | 作用 |
|------|------|------|
| `handlePlanModeTransition` | `bootstrap/state.ts:1349` | 管理plan模式过渡标志 |
| `prepareContextForPlanMode` | `utils/permissions/permissionSetup.ts:1462` | 准备plan模式权限上下文 |
| `getPlanFilePath` | `utils/plans.ts:119` | 获取计划文件路径 |
| `getPlan` | `utils/plans.ts:135` | 读取计划内容 |
| `getPlanSlug` | `utils/plans.ts:32` | 获取/生成计划slug |
| `isPlanModeInterviewPhaseEnabled` | `utils/planModeV2.ts:50` | 检查Interview Phase启用状态 |
| `stripDangerousPermissionsForAutoMode` | `utils/permissions/permissionSetup.ts:510` | 剥离auto模式危险权限 |
| `restoreDangerousPermissions` | `utils/permissions/permissionSetup.ts:561` | 恢复危险权限 |

### B. 状态标志说明

| 标志 | 文件 | 说明 |
|------|------|------|
| `needsPlanModeExitAttachment` | `bootstrap/state.ts` | 需要显示plan模式退出通知 |
| `hasExitedPlanMode` | `bootstrap/state.ts` | 记录是否已退出过plan模式 |
| `prePlanMode` | `ToolPermissionContext` | 进入plan前的模式，用于退出恢复 |
| `strippedDangerousRules` | `ToolPermissionContext` | 被剥离的危险权限规则 |

### C. 环境变量

| 变量 | 说明 |
|------|------|
| `CLAUDE_CODE_PLAN_MODE_INTERVIEW_PHASE` | 启用Interview Phase |
| `CLAUDE_CODE_PLAN_V2_AGENT_COUNT` | Plan模式Agent数量(1-10) |
| `CLAUDE_CODE_PLAN_V2_EXPLORE_AGENT_COUNT` | Explore Agent数量 |

---

*分析完成: 2026-04-02*