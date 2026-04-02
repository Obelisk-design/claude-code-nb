# /clear 命令详解

**分析日期:** 2026-04-02

## 目录

1. [概述](#1-概述)
2. [命令目录逐行分析](#2-命令目录逐行分析)
3. [清理范围详解](#3-清理范围详解)
4. [相关工具和函数](#4-相关工具和函数)
5. [从零实现指南](#5-从零实现指南)

---

## 1. 概述

### 1.1 `/clear` 命令的作用

`/clear` 命令是 Claude Code CLI 中的会话重置命令，用于：

- **清空会话历史**：删除当前会话的所有消息记录
- **重置会话状态**：生成新的会话ID，清理所有会话相关缓存
- **释放上下文空间**：为新的对话提供干净的上下文环境
- **保留特定状态**：有选择性地保留后台任务和关键状态

### 1.2 清理范围概览

```
清理范围分类：
├── 会话数据
│   ├── 消息历史（完全清空）
│   ├── 会话ID（重新生成）
│   └── 会话元数据（标题、标签等）
│
├── 缓存数据
│   ├── 上下文缓存（用户/系统/git状态）
│   ├── 文件建议缓存（@提及）
│   ├── 命令/技能缓存
│   ├── LSP诊断状态
│   └── WebFetch URL缓存（最多50MB）
│
├── 状态重置
│   ├── 工作目录（恢复到原始目录）
│   ├── 文件状态追踪
│   ├── MCP状态（重新初始化）
│   └── 归因状态
│
└── 保留项
    ├── 后台任务（isBackgrounded !== false）
    ├── 工作树状态
    └── 协调器模式状态
```

---

## 2. 命令目录逐行分析

### 2.1 `commands/clear/index.ts` - 命令入口

**文件路径:** `commands/clear/index.ts`

```typescript
import type { LocalCommandCall } from '../../types/command.js'
import { clearConversation } from './conversation.js'

export const call: LocalCommandCall = async (_, context) => {
  await clearConversation(context)
  return { type: 'text', value: '' }
}
```

**逐行解析:**

- **第1行**: 导入 `LocalCommandCall` 类型，定义本地命令的调用签名
- **第2行**: 导入核心清理函数 `clearConversation`
- **第4-7行**: 导出 `call` 函数作为命令处理器
  - `_` 参数：命令参数（未使用）
  - `context` 参数：包含会话状态和操作方法
  - 调用 `clearConversation` 执行清理
  - 返回空文本（不显示任何输出）

**设计要点:**
- 入口文件保持极简，仅作为命令调度器
- 实际逻辑委托给 `conversation.ts`，便于代码复用
- 返回空字符串，用户体验简洁

---

### 2.2 `commands/clear/clear.ts` - 命令定义

**文件路径:** `commands/clear/clear.ts`

```typescript
/**
 * Clear command - minimal metadata only.
 * Implementation is lazy-loaded from clear.ts to reduce startup time.
 * Utility functions:
 * - clearSessionCaches: import from './clear/caches.js'
 * - clearConversation: import from './clear/conversation.js'
 */
import type { Command } from '../../commands.js'

const clear = {
  type: 'local',
  name: 'clear',
  description: 'Clear conversation history and free up context',
  aliases: ['reset', 'new'],
  supportsNonInteractive: false, // Should just create a new session
  load: () => import('./clear.js'),
} satisfies Command

export default clear
```

**逐行解析:**

- **第1-7行**: 文档注释，说明：
  - 命令定义仅包含元数据
  - 实现延迟加载以减少启动时间
  - 提供工具函数的导入路径

- **第8行**: 导入 `Command` 类型定义

- **第10-17行**: 定义命令对象
  - `type: 'local'`: 本地命令，不需要远程调用
  - `name: 'clear'`: 命令名称
  - `description`: 用户可见的描述
  - `aliases: ['reset', 'new']`: 别名，提供多种调用方式
  - `supportsNonInteractive: false`: 非交互模式下应创建新会话
  - `load: () => import('./clear.js')`: 延迟加载实现

- **第18行**: `satisfies Command` 确保类型安全

**设计模式:**
- **延迟加载模式**: 减少启动时的模块加载开销
- **元数据分离**: 命令定义与实现分离，便于维护
- **别名机制**: 提供更符合用户习惯的命令名称

---

### 2.3 `commands/clear/caches.ts` - 缓存清理

**文件路径:** `commands/clear/caches.ts`

#### 2.3.1 导入依赖分析

```typescript
/**
 * Session cache clearing utilities.
 * This module is imported at startup by main.tsx, so keep imports minimal.
 */
import { feature } from 'bun:bundle'
import {
  clearInvokedSkills,
  setLastEmittedDate,
} from '../../bootstrap/state.js'
import { clearCommandsCache } from '../../commands.js'
import { getSessionStartDate } from '../../constants/common.js'
import {
  getGitStatus,
  getSystemContext,
  getUserContext,
  setSystemPromptInjection,
} from '../../context.js'
import { clearFileSuggestionCaches } from '../../hooks/fileSuggestions.js'
import { clearAllPendingCallbacks } from '../../hooks/useSwarmPermissionPoller.js'
import { clearAllDumpState } from '../../services/api/dumpPrompts.js'
import { resetPromptCacheBreakDetection } from '../../services/api/promptCacheBreakDetection.js'
import { clearAllSessions } from '../../services/api/sessionIngress.js'
import { runPostCompactCleanup } from '../../services/compact/postCompactCleanup.js'
import { resetAllLSPDiagnosticState } from '../../services/lsp/LSPDiagnosticRegistry.js'
import { clearTrackedMagicDocs } from '../../services/MagicDocs/magicDocs.js'
import { clearDynamicSkills } from '../../skills/loadSkillsDir.js'
import { resetSentSkillNames } from '../../utils/attachments.js'
import { clearCommandPrefixCaches } from '../../utils/bash/commands.js'
import { resetGetMemoryFilesCache } from '../../utils/claudemd.js'
import { clearRepositoryCaches } from '../../utils/detectRepository.js'
import { clearResolveGitDirCache } from '../../utils/git/gitFilesystem.js'
import { clearStoredImagePaths } from '../../utils/imageStore.js'
import { clearSessionEnvVars } from '../../utils/sessionEnvVars.js'
```

**依赖分类:**

| 类别 | 模块 | 用途 |
|------|------|------|
| 状态管理 | `bootstrap/state.js` | 清理已调用技能、设置最后发出日期 |
| 命令系统 | `commands.js` | 清理命令缓存 |
| 上下文 | `context.js` | 清理用户/系统/git状态缓存 |
| 文件建议 | `hooks/fileSuggestions.js` | 清理@提及的文件建议缓存 |
| 权限管理 | `hooks/useSwarmPermissionPoller.js` | 清理待处理的权限回调 |
| API相关 | `services/api/*` | 清理dump状态、缓存检测、会话入口 |
| 压缩清理 | `services/compact/postCompactCleanup.js` | 运行压缩后清理 |
| LSP | `services/lsp/LSPDiagnosticRegistry.js` | 重置LSP诊断状态 |
| 技能 | `skills/loadSkillsDir.js` | 清理动态技能 |
| Git | `utils/git/*` | 清理Git相关缓存 |

#### 2.3.2 `clearSessionCaches` 函数详解

```typescript
/**
 * Clear all session-related caches.
 * Call this when resuming a session to ensure fresh file/skill discovery.
 * This is a subset of what clearConversation does - it only clears caches
 * without affecting messages, session ID, or triggering hooks.
 *
 * @param preservedAgentIds - Agent IDs whose per-agent state should survive
 *   the clear (e.g., background tasks preserved across /clear). When non-empty,
 *   agentId-keyed state (invoked skills) is selectively cleared and requestId-keyed
 *   state (pending permission callbacks, dump state, cache-break tracking) is left
 *   intact since it cannot be safely scoped to the main session.
 */
export function clearSessionCaches(
  preservedAgentIds: ReadonlySet<string> = new Set(),
): void {
```

**参数说明:**
- `preservedAgentIds`: 需要保留状态的Agent ID集合
- 默认空集合表示全部清理
- 非空时选择性清理Agent相关状态

**清理流程（第50-144行）:**

```typescript
  const hasPreserved = preservedAgentIds.size > 0
  // 1. 清理上下文缓存
  getUserContext.cache.clear?.()
  getSystemContext.cache.clear?.()
  getGitStatus.cache.clear?.()
  getSessionStartDate.cache.clear?.()

  // 2. 清理文件建议缓存（@提及）
  clearFileSuggestionCaches()

  // 3. 清理命令/技能缓存
  clearCommandsCache()

  // 4. 条件清理：仅在没有保留Agent时执行
  if (!hasPreserved) resetPromptCacheBreakDetection()

  // 5. 清理系统提示注入（缓存破坏器）
  setSystemPromptInjection(null)

  // 6. 重置最后发出日期
  setLastEmittedDate(null)

  // 7. 运行压缩后清理
  runPostCompactCleanup()

  // 8. 重置已发送技能名称
  resetSentSkillNames()

  // 9. 重置内存文件缓存
  resetGetMemoryFilesCache('session_start')

  // 10. 清理存储的图片路径缓存
  clearStoredImagePaths()

  // 11. 清理会话入口缓存
  clearAllSessions()

  // 12. 条件清理：权限回调
  if (!hasPreserved) clearAllPendingCallbacks()

  // 13. Tungsten会话使用追踪（仅ant用户）
  if (process.env.USER_TYPE === 'ant') {
    void import('../../tools/TungstenTool/TungstenTool.js').then(
      ({ clearSessionsWithTungstenUsage, resetInitializationState }) => {
        clearSessionsWithTungstenUsage()
        resetInitializationState()
      },
    )
  }

  // 14. 归因缓存（条件特性标志）
  if (feature('COMMIT_ATTRIBUTION')) {
    void import('../../utils/attributionHooks.js').then(
      ({ clearAttributionCaches }) => clearAttributionCaches(),
    )
  }

  // 15. 清理仓库检测缓存
  clearRepositoryCaches()

  // 16. 清理bash命令前缀缓存
  clearCommandPrefixCaches()

  // 17. 条件清理：dump提示状态
  if (!hasPreserved) clearAllDumpState()

  // 18. 清理已调用技能（带保留过滤）
  clearInvokedSkills(preservedAgentIds)

  // 19. 清理Git目录解析缓存
  clearResolveGitDirCache()

  // 20. 清理动态技能
  clearDynamicSkills()

  // 21. 重置LSP诊断状态
  resetAllLSPDiagnosticState()

  // 22. 清理追踪的Magic文档
  clearTrackedMagicDocs()

  // 23. 清理会话环境变量
  clearSessionEnvVars()

  // 24. 清理WebFetch URL缓存（最多50MB）
  void import('../../tools/WebFetchTool/utils.js').then(
    ({ clearWebFetchCache }) => clearWebFetchCache(),
  )

  // 25. 清理工具搜索描述缓存
  void import('../../tools/ToolSearchTool/ToolSearchTool.js').then(
    ({ clearToolSearchDescriptionCache }) => clearToolSearchDescriptionCache(),
  )

  // 26. 清理Agent定义缓存
  void import('../../tools/AgentTool/loadAgentsDir.js').then(
    ({ clearAgentDefinitionsCache }) => clearAgentDefinitionsCache(),
  )

  // 27. 清理技能工具提示缓存
  void import('../../tools/SkillTool/prompt.js').then(({ clearPromptCache }) =>
    clearPromptCache(),
  )
}
```

**设计要点:**

1. **条件清理策略**: 根据 `hasPreserved` 决定是否清理Agent无关的全局状态

2. **动态导入优化**: 大型模块使用 `void import().then()` 异步加载，避免阻塞主流程

3. **特性标志检查**: `feature('COMMIT_ATTRIBUTION')` 等条件编译优化

4. **缓存清理顺序**: 从核心到外围，先清理基础缓存再清理依赖缓存

---

### 2.4 `commands/clear/conversation.ts` - 会话清理

**文件路径:** `commands/clear/conversation.ts`

#### 2.4.1 导入依赖分析

```typescript
import { feature } from 'bun:bundle'
import { randomUUID, type UUID } from 'crypto'
import {
  getLastMainRequestId,
  getOriginalCwd,
  getSessionId,
  regenerateSessionId,
} from '../../bootstrap/state.js'
import {
  type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS,
  logEvent,
} from '../../services/analytics/index.js'
import type { AppState } from '../../state/AppState.js'
import { isInProcessTeammateTask } from '../../tasks/InProcessTeammateTask/types.js'
import {
  isLocalAgentTask,
  type LocalAgentTaskState,
} from '../../tasks/LocalAgentTask/LocalAgentTask.js'
import { isLocalShellTask } from '../../tasks/LocalShellTask/guards.js'
import { asAgentId } from '../../types/ids.js'
import type { Message } from '../../types/message.js'
import { createEmptyAttributionState } from '../../utils/commitAttribution.js'
import type { FileStateCache } from '../../utils/fileStateCache.js'
import {
  executeSessionEndHooks,
  getSessionEndHookTimeoutMs,
} from '../../utils/hooks.js'
import { logError } from '../../utils/log.js'
import { clearAllPlanSlugs } from '../../utils/plans.js'
import { setCwd } from '../../utils/Shell.js'
import { processSessionStartHooks } from '../../utils/sessionStart.js'
import {
  clearSessionMetadata,
  getAgentTranscriptPath,
  resetSessionFilePointer,
  saveWorktreeState,
} from '../../utils/sessionStorage.js'
import {
  evictTaskOutput,
  initTaskOutputAsSymlink,
} from '../../utils/task/diskOutput.js'
import { getCurrentWorktreeSession } from '../../utils/worktree.js'
import { clearSessionCaches } from './caches.js'
```

**依赖分类:**

| 类别 | 模块 | 用途 |
|------|------|------|
| 核心状态 | `bootstrap/state.js` | 会话ID生成、原始目录获取 |
| 分析 | `services/analytics/` | 事件日志记录 |
| 任务管理 | `tasks/*` | 任务状态判断和清理 |
| Hooks | `utils/hooks.js`, `utils/sessionStart.js` | 会话生命周期钩子 |
| 存储 | `utils/sessionStorage.js` | 会话元数据和文件指针管理 |
| 工作树 | `utils/worktree.js` | 工作树状态管理 |

#### 2.4.2 `clearConversation` 函数详解

```typescript
export async function clearConversation({
  setMessages,
  readFileState,
  discoveredSkillNames,
  loadedNestedMemoryPaths,
  getAppState,
  setAppState,
  setConversationId,
}: {
  setMessages: (updater: (prev: Message[]) => Message[]) => void
  readFileState: FileStateCache
  discoveredSkillNames?: Set<string>
  loadedNestedMemoryPaths?: Set<string>
  getAppState?: () => AppState
  setAppState?: (f: (prev: AppState) => AppState) => void
  setConversationId?: (id: UUID) => void
}): Promise<void> {
```

**参数说明:**

| 参数 | 类型 | 用途 |
|------|------|------|
| `setMessages` | Function | 更新消息列表的函数 |
| `readFileState` | FileStateCache | 文件状态缓存 |
| `discoveredSkillNames` | Set<string> | 已发现的技能名称 |
| `loadedNestedMemoryPaths` | Set<string> | 已加载的嵌套内存路径 |
| `getAppState` | Function | 获取应用状态 |
| `setAppState` | Function | 更新应用状态 |
| `setConversationId` | Function | 更新会话ID（用于UI重渲染） |

**清理流程详解:**

##### 步骤1: 执行SessionEnd钩子（第66-74行）

```typescript
  // Execute SessionEnd hooks before clearing (bounded by
  // CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS, default 1.5s)
  const sessionEndTimeoutMs = getSessionEndHookTimeoutMs()
  await executeSessionEndHooks('clear', {
    getAppState,
    setAppState,
    signal: AbortSignal.timeout(sessionEndTimeoutMs),
    timeoutMs: sessionEndTimeoutMs,
  })
```

**关键点:**
- 清理前执行用户定义的钩子
- 默认超时1.5秒，防止钩子阻塞
- 使用 `AbortSignal.timeout` 实现超时控制

##### 步骤2: 发送缓存驱逐提示（第76-85行）

```typescript
  // Signal to inference that this conversation's cache can be evicted.
  const lastRequestId = getLastMainRequestId()
  if (lastRequestId) {
    logEvent('tengu_cache_eviction_hint', {
      scope:
        'conversation_clear' as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS,
      last_request_id:
        lastRequestId as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS,
    })
  }
```

**目的:** 通知推理服务可以驱逐此会话的缓存

##### 步骤3: 计算保留的任务（第87-107行）

```typescript
  // Compute preserved tasks up front so their per-agent state survives the
  // cache wipe below. A task is preserved unless it explicitly has
  // isBackgrounded === false. Main-session tasks (Ctrl+B) are preserved —
  // they write to an isolated per-task transcript and run under an agent
  // context, so they're safe across session ID regeneration.
  const preservedAgentIds = new Set<string>()
  const preservedLocalAgents: LocalAgentTaskState[] = []
  const shouldKillTask = (task: AppState['tasks'][string]): boolean =>
    'isBackgrounded' in task && task.isBackgrounded === false
  if (getAppState) {
    for (const task of Object.values(getAppState().tasks)) {
      if (shouldKillTask(task)) continue
      if (isLocalAgentTask(task)) {
        preservedAgentIds.add(task.agentId)
        preservedLocalAgents.push(task)
      } else if (isInProcessTeammateTask(task)) {
        preservedAgentIds.add(task.identity.agentId)
      }
    }
  }
```

**保留策略:**
- `isBackgrounded !== false` 的任务被保留
- 后台任务（Ctrl+B）写入独立的转录文件，跨会话安全
- 前台任务（`isBackgrounded === false`）被终止

##### 步骤4: 清空消息列表（第109行）

```typescript
  setMessages(() => [])
```

##### 步骤5: 重置上下文阻塞标志（第111-117行）

```typescript
  // Clear context-blocked flag so proactive ticks resume after /clear
  if (feature('PROACTIVE') || feature('KAIROS')) {
    /* eslint-disable @typescript-eslint/no-require-imports */
    const { setContextBlocked } = require('../../proactive/index.js')
    /* eslint-enable @typescript-eslint/no-require-imports */
    setContextBlocked(false)
  }
```

##### 步骤6: 强制Logo重渲染（第119-122行）

```typescript
  // Force logo re-render by updating conversationId
  if (setConversationId) {
    setConversationId(randomUUID())
  }
```

##### 步骤7: 清理会话缓存（第124-127行）

```typescript
  // Clear all session-related caches. Per-agent state for preserved background
  // tasks (invoked skills, pending permission callbacks, dump state, cache-break
  // tracking) is retained so those agents keep functioning.
  clearSessionCaches(preservedAgentIds)
```

##### 步骤8: 重置工作目录和状态（第129-132行）

```typescript
  setCwd(getOriginalCwd())
  readFileState.clear()
  discoveredSkillNames?.clear()
  loadedNestedMemoryPaths?.clear()
```

##### 步骤9: 更新应用状态（第134-192行）

```typescript
  // Clean out necessary items from App State
  if (setAppState) {
    setAppState(prev => {
      // Partition tasks using the same predicate computed above:
      // kill+remove foreground tasks, preserve everything else.
      const nextTasks: AppState['tasks'] = {}
      for (const [taskId, task] of Object.entries(prev.tasks)) {
        if (!shouldKillTask(task)) {
          nextTasks[taskId] = task
          continue
        }
        // Foreground task: kill it and drop from state
        try {
          if (task.status === 'running') {
            if (isLocalShellTask(task)) {
              task.shellCommand?.kill()
              task.shellCommand?.cleanup()
              if (task.cleanupTimeoutId) {
                clearTimeout(task.cleanupTimeoutId)
              }
            }
            if ('abortController' in task) {
              task.abortController?.abort()
            }
            if ('unregisterCleanup' in task) {
              task.unregisterCleanup?.()
            }
          }
        } catch (error) {
          logError(error)
        }
        void evictTaskOutput(taskId)
      }

      return {
        ...prev,
        tasks: nextTasks,
        attribution: createEmptyAttributionState(),
        // Clear standalone agent context (name/color set by /rename, /color)
        // so the new session doesn't display the old session's identity badge
        standaloneAgentContext: undefined,
        fileHistory: {
          snapshots: [],
          trackedFiles: new Set(),
          snapshotSequence: 0,
        },
        // Reset MCP state to default to trigger re-initialization.
        // Preserve pluginReconnectKey so /clear doesn't cause a no-op
        // (it's only bumped by /reload-plugins).
        mcp: {
          clients: [],
          tools: [],
          commands: [],
          resources: {},
          pluginReconnectKey: prev.mcp.pluginReconnectKey,
        },
      }
    })
  }
```

**状态重置内容:**
- 终止前台任务
- 清空归因状态
- 清除独立Agent上下文
- 重置文件历史
- 重置MCP状态（保留 `pluginReconnectKey`）

##### 步骤10: 清理计划Slug缓存（第194-195行）

```typescript
  // Clear plan slug cache so a new plan file is used after /clear
  clearAllPlanSlugs()
```

##### 步骤11: 清理会话元数据（第197-199行）

```typescript
  // Clear cached session metadata (title, tag, agent name/color)
  // so the new session doesn't inherit the previous session's identity
  clearSessionMetadata()
```

##### 步骤12: 重新生成会话ID（第201-208行）

```typescript
  // Generate new session ID to provide fresh state
  // Set the old session as parent for analytics lineage tracking
  regenerateSessionId({ setCurrentAsParent: true })
  // Update the environment variable so subprocesses use the new session ID
  if (process.env.USER_TYPE === 'ant' && process.env.CLAUDE_CODE_SESSION_ID) {
    process.env.CLAUDE_CODE_SESSION_ID = getSessionId()
  }
  await resetSessionFilePointer()
```

##### 步骤13: 重新指向保留任务的输出（第210-224行）

```typescript
  // Preserved local_agent tasks had their TaskOutput symlink baked against the
  // old session ID at spawn time, but post-clear transcript writes land under
  // the new session directory (appendEntry re-reads getSessionId()). Re-point
  // the symlinks so TaskOutput reads the live file instead of a frozen pre-clear
  // snapshot. Only re-point running tasks — finished tasks will never write
  // again, so re-pointing would replace a valid symlink with a dangling one.
  for (const task of preservedLocalAgents) {
    if (task.status !== 'running') continue
    void initTaskOutputAsSymlink(
      task.id,
      getAgentTranscriptPath(asAgentId(task.agentId)),
    )
  }
```

##### 步骤14: 持久化模式和工作树状态（第226-242行）

```typescript
  // Re-persist mode and worktree state after the clear so future --resume
  // knows what the new post-clear session was in. clearSessionMetadata
  // wiped both from the cache, but the process is still in the same mode
  // and (if applicable) the same worktree directory.
  if (feature('COORDINATOR_MODE')) {
    /* eslint-disable @typescript-eslint/no-require-imports */
    const { saveMode } = require('../../utils/sessionStorage.js')
    const {
      isCoordinatorMode,
    } = require('../../coordinator/coordinatorMode.js')
    /* eslint-enable @typescript-eslint/no-require-imports */
    saveMode(isCoordinatorMode() ? 'coordinator' : 'normal')
  }
  const worktreeSession = getCurrentWorktreeSession()
  if (worktreeSession) {
    saveWorktreeState(worktreeSession)
  }
```

##### 步骤15: 执行SessionStart钩子（第244-250行）

```typescript
  // Execute SessionStart hooks after clearing
  const hookMessages = await processSessionStartHooks('clear')

  // Update messages with hook results
  if (hookMessages.length > 0) {
    setMessages(() => hookMessages)
  }
}
```

---

## 3. 清理范围详解

### 3.1 会话历史清理

**涉及的缓存/状态:**

| 缓存类型 | 清理方式 | 影响范围 |
|---------|---------|---------|
| 消息列表 | `setMessages(() => [])` | 清空所有对话历史 |
| 会话ID | `regenerateSessionId()` | 生成新ID，旧ID设为父ID |
| 会话元数据 | `clearSessionMetadata()` | 标题、标签、Agent名称/颜色 |
| 计划Slug | `clearAllPlanSlugs()` | 清除计划文件关联 |

### 3.2 缓存清理详解

#### 3.2.1 上下文缓存

```typescript
// 用户上下文缓存
getUserContext.cache.clear?.()

// 系统上下文缓存
getSystemContext.cache.clear?.()

// Git状态缓存
getGitStatus.cache.clear?.()

// 会话开始日期缓存
getSessionStartDate.cache.clear?.()
```

**作用:** 强制重新计算上下文信息

#### 3.2.2 文件建议缓存

```typescript
clearFileSuggestionCaches()  // @提及的文件建议
```

**作用:** 清除文件建议索引，下次@提及时重新扫描

#### 3.2.3 命令/技能缓存

```typescript
clearCommandsCache()         // 已发现的命令缓存
clearDynamicSkills()         // 动态加载的技能
resetSentSkillNames()        // 已发送的技能名称列表
```

**作用:** 强制重新发现命令和技能

#### 3.2.4 LSP诊断状态

```typescript
resetAllLSPDiagnosticState()
```

**作用:** 清除LSP诊断追踪，防止旧的诊断信息残留

#### 3.2.5 WebFetch缓存

```typescript
void import('../../tools/WebFetchTool/utils.js').then(
  ({ clearWebFetchCache }) => clearWebFetchCache(),
)
```

**作用:** 清除最多50MB的网页内容缓存

### 3.3 配置清理

**不清理的配置:**

| 配置类型 | 原因 |
|---------|------|
| 设置文件 | 用户配置应持久化 |
| 环境变量 | 仅清理会话级环境变量 |
| 项目根目录 | 启动时固定，不应改变 |

**清理的会话配置:**

```typescript
clearSessionEnvVars()  // 清理会话级环境变量
```

### 3.4 MCP状态清理

```typescript
mcp: {
  clients: [],
  tools: [],
  commands: [],
  resources: {},
  pluginReconnectKey: prev.mcp.pluginReconnectKey,  // 保留！
}
```

**保留 `pluginReconnectKey` 的原因:**
- `pluginReconnectKey` 仅由 `/reload-plugins` 更新
- 保留它可以防止 `/clear` 触发无意义的重连

---

## 4. 相关工具和函数

### 4.1 `utils/sessionStorage.ts` 相关函数

#### 4.1.1 `clearSessionMetadata`

**文件路径:** `utils/sessionStorage.ts:2792-2805`

```typescript
export function clearSessionMetadata(): void {
  const project = getProject()
  project.currentSessionTitle = undefined
  project.currentSessionTag = undefined
  project.currentSessionAgentName = undefined
  project.currentSessionAgentColor = undefined
  project.currentSessionLastPrompt = undefined
  project.currentSessionAgentSetting = undefined
  project.currentSessionMode = undefined
  project.currentSessionWorktree = undefined
  project.currentSessionPrNumber = undefined
  project.currentSessionPrUrl = undefined
  project.currentSessionPrRepository = undefined
}
```

**作用:** 清除当前会话的元数据缓存

#### 4.1.2 `resetSessionFilePointer`

**文件路径:** `utils/sessionStorage.ts:1505-1507`

```typescript
export async function resetSessionFilePointer() {
  getProject().resetSessionFile()
}
```

**作用:** 重置会话文件指针，使下次写入创建新文件

#### 4.1.3 `saveWorktreeState`

**文件路径:** `utils/sessionStorage.ts:2889-2920`

```typescript
export function saveWorktreeState(
  worktreeSession: PersistedWorktreeSession | null,
): void {
  // Strip ephemeral fields (creationDurationMs, usedSparsePaths)
  const stripped: PersistedWorktreeSession | null = worktreeSession
    ? {
        originalCwd: worktreeSession.originalCwd,
        worktreePath: worktreeSession.worktreePath,
        worktreeName: worktreeSession.worktreeName,
        worktreeBranch: worktreeSession.worktreeBranch,
        originalBranch: worktreeSession.originalBranch,
        originalHeadCommit: worktreeSession.originalHeadCommit,
        sessionId: worktreeSession.sessionId,
        tmuxSessionName: worktreeSession.tmuxSessionName,
        hookBased: worktreeSession.hookBased,
      }
    : null
  const project = getProject()
  project.currentSessionWorktree = stripped
  if (project.sessionFile) {
    appendEntryToFile(project.sessionFile, {
      type: 'worktree-state',
      worktreeSession: stripped,
      sessionId: getSessionId(),
    })
  }
}
```

**作用:** 保存工作树状态以便 `--resume` 恢复

#### 4.1.4 `getAgentTranscriptPath`

**文件路径:** `utils/sessionStorage.ts:247-258`

```typescript
export function getAgentTranscriptPath(agentId: AgentId): string {
  const projectDir = getSessionProjectDir() ?? getProjectDir(getOriginalCwd())
  const sessionId = getSessionId()
  const subdir = agentTranscriptSubdirs.get(agentId)
  const base = subdir
    ? join(projectDir, sessionId, 'subagents', subdir)
    : join(projectDir, sessionId, 'subagents')
  return join(base, `agent-${agentId}.jsonl`)
}
```

**作用:** 获取Agent转录文件路径

### 4.2 `bootstrap/state.ts` 相关函数

#### 4.2.1 `getSessionId` 和 `regenerateSessionId`

**文件路径:** `bootstrap/state.ts:431-450`

```typescript
export function getSessionId(): SessionId {
  return STATE.sessionId
}

export function regenerateSessionId(
  options: { setCurrentAsParent?: boolean } = {},
): SessionId {
  if (options.setCurrentAsParent) {
    STATE.parentSessionId = STATE.sessionId
  }
  // Drop the outgoing session's plan-slug entry
  STATE.planSlugCache.delete(STATE.sessionId)
  // Regenerated sessions live in the current project
  STATE.sessionId = randomUUID() as SessionId
  STATE.sessionProjectDir = null
  return STATE.sessionId
}
```

**作用:**
- `getSessionId`: 获取当前会话ID
- `regenerateSessionId`: 生成新会话ID，可选择设置父会话ID

#### 4.2.2 `getLastMainRequestId`

**文件路径:** `bootstrap/state.ts:753-755`

```typescript
export function getLastMainRequestId(): string | undefined {
  return STATE.lastMainRequestId
}
```

**作用:** 获取最后的请求ID，用于缓存驱逐提示

#### 4.2.3 `clearInvokedSkills`

**文件路径:** `bootstrap/state.ts:1543-1555`

```typescript
export function clearInvokedSkills(
  preservedAgentIds?: ReadonlySet<string>,
): void {
  if (!preservedAgentIds || preservedAgentIds.size === 0) {
    STATE.invokedSkills.clear()
    return
  }
  for (const [key, skill] of STATE.invokedSkills) {
    if (skill.agentId === null || !preservedAgentIds.has(skill.agentId)) {
      STATE.invokedSkills.delete(key)
    }
  }
}
```

**作用:** 清理已调用技能缓存，支持选择性保留

#### 4.2.4 `setLastEmittedDate`

**文件路径:** `bootstrap/state.ts:1662-1664`

```typescript
export function setLastEmittedDate(date: string | null): void {
  STATE.lastEmittedDate = date
}
```

**作用:** 设置最后发出日期，用于事件去重

### 4.3 `utils/settings/settingsCache.ts` 相关函数

**文件路径:** `utils/settings/settingsCache.ts`

```typescript
// 会话设置缓存
let sessionSettingsCache: SettingsWithErrors | null = null

// 每源缓存
const perSourceCache = new Map<SettingSource, SettingsJson | null>()

// 解析文件缓存
const parseFileCache = new Map<string, ParsedSettings>()

// 插件设置基座
let pluginSettingsBase: Record<string, unknown> | undefined

export function resetSettingsCache(): void {
  sessionSettingsCache = null
  perSourceCache.clear()
  parseFileCache.clear()
}
```

**注意:** `/clear` 命令**不**调用 `resetSettingsCache()`，用户设置是持久化的。

---

## 5. 从零实现指南

### 5.1 如何设计清理命令

#### 步骤1: 定义命令结构

```typescript
// commands/clear/index.ts
import type { LocalCommandCall } from '../../types/command.js'

export const call: LocalCommandCall = async (_, context) => {
  // 1. 执行清理前钩子
  await executePreClearHooks(context)

  // 2. 清理消息历史
  context.setMessages(() => [])

  // 3. 清理缓存
  clearAllCaches()

  // 4. 重置状态
  resetSessionState()

  // 5. 执行清理后钩子
  await executePostClearHooks(context)

  return { type: 'text', value: '' }
}
```

#### 步骤2: 实现缓存清理

```typescript
// commands/clear/caches.ts
export function clearSessionCaches(
  preservedAgentIds: ReadonlySet<string> = new Set(),
): void {
  // 按依赖顺序清理
  // 1. 核心上下文
  clearContextCaches()

  // 2. 文件系统
  clearFileCaches()

  // 3. 工具状态
  clearToolCaches()

  // 4. 条件清理（保留后台任务）
  if (preservedAgentIds.size === 0) {
    clearGlobalState()
  } else {
    clearSelectiveState(preservedAgentIds)
  }
}
```

### 5.2 安全清理策略

#### 原则1: 先备份后清理

```typescript
// 保存需要持久化的状态
const preservedState = {
  worktreeSession: getCurrentWorktreeSession(),
  mode: isCoordinatorMode() ? 'coordinator' : 'normal',
}

// 执行清理
clearAll()

// 恢复持久化状态
if (preservedState.worktreeSession) {
  saveWorktreeState(preservedState.worktreeSession)
}
```

#### 原则2: 使用超时保护

```typescript
// 钩子执行超时保护
const timeout = getSessionEndHookTimeoutMs()  // 默认1.5秒
await executeSessionEndHooks('clear', {
  signal: AbortSignal.timeout(timeout),
  timeoutMs: timeout,
})
```

#### 原则3: 条件清理

```typescript
// 根据保留ID选择性清理
if (preservedAgentIds.size > 0) {
  // 仅清理不属于保留Agent的状态
  for (const [key, value] of state.entries()) {
    if (!preservedAgentIds.has(value.agentId)) {
      state.delete(key)
    }
  }
} else {
  // 清理所有状态
  state.clear()
}
```

### 5.3 用户确认机制

#### 实现确认对话框

```typescript
export async function clearConversation(context: ClearContext): Promise<void> {
  // 检查是否有活动任务
  const activeTasks = Object.values(context.getAppState().tasks)
    .filter(t => t.status === 'running')

  if (activeTasks.length > 0) {
    const confirmed = await confirmWithUser({
      message: `有 ${activeTasks.length} 个正在运行的任务将被终止。继续吗？`,
      defaultAction: 'cancel',
    })

    if (!confirmed) {
      return
    }
  }

  // 继续清理
  await doClear(context)
}
```

#### 后台任务保护

```typescript
// 计算保留的任务
const preservedAgentIds = new Set<string>()

for (const task of Object.values(getAppState().tasks)) {
  // 仅终止明确标记为前台的任务
  if (task.isBackgrounded === false) {
    continue  // 跳过，稍后终止
  }

  // 后台任务保留其Agent ID
  if (task.agentId) {
    preservedAgentIds.add(task.agentId)
  }
}
```

### 5.4 完整实现示例

```typescript
// commands/clear/clearImpl.ts

interface ClearOptions {
  setMessages: (updater: (prev: Message[]) => Message[]) => void
  readFileState: FileStateCache
  getAppState?: () => AppState
  setAppState?: (f: (prev: AppState) => AppState) => AppState
}

export async function clearConversation(options: ClearOptions): Promise<void> {
  const { setMessages, readFileState, getAppState, setAppState } = options

  try {
    // 1. 执行SessionEnd钩子
    const timeout = getSessionEndHookTimeoutMs()
    await executeSessionEndHooks('clear', {
      getAppState,
      setAppState,
      signal: AbortSignal.timeout(timeout),
      timeoutMs: timeout,
    })

    // 2. 计算保留的任务
    const preservedAgentIds = computePreservedAgents(getAppState)

    // 3. 清空消息
    setMessages(() => [])

    // 4. 清理缓存
    clearSessionCaches(preservedAgentIds)

    // 5. 重置状态
    if (setAppState) {
      setAppState(prev => ({
        ...prev,
        tasks: filterTasks(prev.tasks, preservedAgentIds),
        attribution: createEmptyAttributionState(),
        fileHistory: createEmptyFileHistory(),
        mcp: resetMCPState(prev.mcp),
      }))
    }

    // 6. 重新生成会话ID
    regenerateSessionId({ setCurrentAsParent: true })

    // 7. 清理会话元数据
    clearSessionMetadata()
    await resetSessionFilePointer()

    // 8. 执行SessionStart钩子
    const hookMessages = await processSessionStartHooks('clear')
    if (hookMessages.length > 0) {
      setMessages(() => hookMessages)
    }

  } catch (error) {
    logError(error)
    throw error
  }
}

function computePreservedAgents(
  getAppState?: () => AppState,
): Set<string> {
  const preserved = new Set<string>()

  if (!getAppState) return preserved

  for (const task of Object.values(getAppState().tasks)) {
    if (task.isBackgrounded !== false && task.agentId) {
      preserved.add(task.agentId)
    }
  }

  return preserved
}
```

### 5.5 测试策略

```typescript
// commands/clear/__tests__/clear.test.ts

describe('clearConversation', () => {
  it('should clear all messages', async () => {
    const setMessages = jest.fn()
    await clearConversation({ setMessages, readFileState: mockCache })

    expect(setMessages).toHaveBeenCalledWith(expect.any(Function))
    const updater = setMessages.mock.calls[0][0]
    expect(updater([mockMessage])).toEqual([])
  })

  it('should preserve background tasks', async () => {
    const mockState = {
      tasks: {
        'bg-task': { isBackgrounded: true, agentId: 'agent-1' },
        'fg-task': { isBackgrounded: false },
      },
    }

    await clearConversation({
      setMessages: jest.fn(),
      readFileState: mockCache,
      getAppState: () => mockState,
      setAppState: jest.fn(),
    })

    expect(clearSessionCaches).toHaveBeenCalledWith(
      new Set(['agent-1'])
    )
  })

  it('should execute hooks with timeout', async () => {
    await clearConversation(mockContext)

    expect(executeSessionEndHooks).toHaveBeenCalledWith(
      'clear',
      expect.objectContaining({
        timeoutMs: 1500,
      })
    )
  })
})
```

---

## 总结

`/clear` 命令是一个复杂的会话重置机制，涉及多个层面的清理：

1. **消息清理**：直接清空消息列表
2. **缓存清理**：清理20+种不同类型的缓存
3. **状态重置**：重置会话ID、工作目录、文件状态
4. **任务管理**：选择性保留后台任务，终止前台任务
5. **钩子执行**：在清理前后执行用户定义的钩子
6. **持久化**：重新保存需要跨会话的状态

**关键设计模式:**

- **选择性清理**: 通过 `preservedAgentIds` 保留后台任务状态
- **延迟加载**: 命令实现延迟加载，减少启动时间
- **超时保护**: 钩子执行有超时限制，防止阻塞
- **顺序保证**: 先执行钩子，再清理，最后恢复状态

这种设计确保了 `/clear` 命令既能有效重置会话，又能保护需要持久化的状态。