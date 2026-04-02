# Resume 会话恢复详解

**分析日期:** 2026-04-02

## 1. 概述

### Resume 命令的作用

`/resume` 命令是 Claude Code CLI 的核心功能之一，用于恢复之前中断的对话会话。它允许用户：

- **恢复上次会话**: 无需参数即可选择最近的历史会话
- **恢复特定会话**: 通过会话 ID 或自定义标题恢复指定会话
- **跨项目恢复**: 支持在不同项目目录间恢复会话
- **分支会话**: 通过 `/branch` 创建会话分支并恢复

### 会话持久化机制

Claude Code 采用 JSONL 格式的转录文件（transcript）进行会话持久化：

- **存储位置**: `~/.claude/projects/<sanitized-project-path>/<session-id>.jsonl`
- **存储格式**: 每行一个 JSON 对象，包含消息、元数据、快照等条目
- **增量写入**: 消息逐条追加，避免全文件重写
- **延迟物化**: 会话文件在首条用户/助手消息时创建，避免空文件

核心存储类 `Project`（位于 `utils/sessionStorage.ts`）管理会话写入：

```typescript
// 写入队列机制 - 避免并发写入冲突
private writeQueues = new Map<string, Array<{ entry: Entry; resolve: () => void }>>()
private flushTimer: ReturnType<typeof setTimeout> | null = null
private FLUSH_INTERVAL_MS = 100  // 100ms 批量写入间隔
```

---

## 2. 命令入口分析

### commands/resume/index.ts

```typescript
const resume: Command = {
  type: 'local-jsx',
  name: 'resume',
  description: 'Resume a previous conversation',
  aliases: ['continue'],  // 别名：/continue
  argumentHint: '[conversation id or search term]',
  load: () => import('./resume.js'),
}
```

**命令参数和选项:**

| 参数类型 | 示例 | 说明 |
|---------|------|------|
| 无参数 | `/resume` | 显示会话选择器 UI |
| 会话 ID | `/resume abc123...` | 直接恢复指定 UUID 的会话 |
| 自定义标题 | `/resume my-session-name` | 搜索并恢复匹配标题的会话 |

### 命令处理流程

`commands/resume/resume.tsx` 中的 `call` 函数是入口点：

```typescript
export const call: LocalJSXCommandCall = async (onDone, context, args) => {
  const arg = args?.trim()

  // 1. 无参数 - 显示选择器
  if (!arg) {
    return <ResumeCommand key={Date.now()} onDone={onDone} onResume={onResume} />
  }

  // 2. 加载同仓库的所有会话日志
  const worktreePaths = await getWorktreePaths(getOriginalCwd())
  const logs = await loadSameRepoMessageLogs(worktreePaths)

  // 3. 尝试作为 UUID 匹配
  const maybeSessionId = validateUuid(arg)
  if (maybeSessionId) {
    // 直接按 ID 查找并恢复
  }

  // 4. 尝试作为自定义标题匹配
  if (isCustomTitleEnabled()) {
    const titleMatches = await searchSessionsByCustomTitle(arg, { exact: true })
    // 精确匹配单个则恢复，多个则报错
  }

  // 5. 未找到 - 显示错误
}
```

---

## 3. 会话存储机制

### utils/sessionStorage.ts 逐行分析

#### 3.1 存储路径计算

```typescript
// 获取项目目录 - 路径经过 sanitize 处理
export const getProjectDir = memoize((projectDir: string): string => {
  return join(getProjectsDir(), sanitizePath(projectDir))
})

// 会话转录文件路径
export function getTranscriptPath(): string {
  const projectDir = getSessionProjectDir() ?? getProjectDir(getOriginalCwd())
  return join(projectDir, `${getSessionId()}.jsonl`)
}
```

**路径示例:**
- 项目路径: `/home/user/my-project`
- 存储路径: `~/.claude/projects/-home-user-my-project/<uuid>.jsonl`

#### 3.2 JSONL 条目类型

```typescript
type Entry =
  | UserMessage           // 用户消息
  | AssistantMessage      // 助手消息
  | AttachmentMessage     // 附件消息
  | SystemMessage         // 系统消息
  | SummaryEntry          // 会话摘要
  | CustomTitleEntry      // 自定义标题
  | TagEntry              // 标签
  | FileHistorySnapshot   // 文件历史快照
  | AttributionSnapshot   // 提交归属快照
  | ContentReplacement    // 内容替换记录
  | ContextCollapseCommit // 上下文折叠提交
  | WorktreeState         // 工作树状态
  | PrLink                // PR 链接
  | ModeEntry             // 模式记录
```

#### 3.3 消息写入机制

```typescript
class Project {
  // 写入队列 - 每个文件独立队列
  private writeQueues = new Map<string, Array<{ entry: Entry; resolve: () => void }>>()

  // 批量写入调度
  private scheduleDrain(): void {
    if (this.flushTimer) return  // 已有待处理的写入
    this.flushTimer = setTimeout(async () => {
      this.flushTimer = null
      this.activeDrain = this.drainWriteQueue()
      await this.activeDrain
    }, this.FLUSH_INTERVAL_MS)  // 100ms 延迟批量写入
  }

  // 实际写入 - 批量追加
  private async drainWriteQueue(): Promise<void> {
    for (const [filePath, queue] of this.writeQueues) {
      const batch = queue.splice(0)
      let content = ''
      for (const { entry, resolve } of batch) {
        content += jsonStringify(entry) + '\n'
      }
      await this.appendToFile(filePath, content)
    }
  }
}
```

#### 3.4 会话日志格式

每条 JSONL 记录包含以下核心字段：

```json
{
  "type": "user",                    // 消息类型
  "uuid": "abc123...",               // 消息唯一标识
  "parentUuid": "prev123...",        // 父消息 UUID（构建消息链）
  "sessionId": "session-uuid",       // 会话 ID
  "timestamp": "2024-01-01T00:00:00Z", // 时间戳
  "message": {                       // 消息内容
    "content": "Hello Claude"
  }
}
```

---

## 4. 会话恢复 UI

### screens/ResumeConversation.tsx 分析

#### 4.1 组件结构

```typescript
export function ResumeConversation({
  commands,
  worktreePaths,
  initialTools,
  mcpClients,
  // ...其他属性
}: Props): React.ReactNode {
  const [logs, setLogs] = React.useState<LogOption[]>([])
  const [loading, setLoading] = React.useState(true)
  const [resuming, setResuming] = React.useState(false)
  const [showAllProjects, setShowAllProjects] = React.useState(false)

  // 进度式加载 - 避免大量文件一次性读取
  const sessionLogResultRef = React.useRef<SessionLogResult | null>(null)
}
```

#### 4.2 会话列表加载

```typescript
React.useEffect(() => {
  loadSameRepoMessageLogsProgressive(worktreePaths).then(result => {
    sessionLogResultRef.current = result
    logCountRef.current = result.logs.length
    setLogs(result.logs)
    setLoading(false)
  })
}, [worktreePaths])
```

**进度式加载机制:**
- `loadSameRepoMessageLogsProgressive` 返回初始批次 + 完整列表引用
- 用户滚动到底部时调用 `loadMoreLogs(count)` 加载更多
- 避免一次性读取数百个会话文件导致延迟

#### 4.3 会话列表展示

使用 `LogSelector` 组件展示会话列表：

```typescript
<LogSelector
  logs={filteredLogs}
  maxHeight={rows}
  onCancel={onCancel}
  onSelect={onSelect}
  onLoadMore={loadMoreLogs}
  showAllProjects={showAllProjects}
  onToggleAllProjects={handleToggleAllProjects}
  onAgenticSearch={agenticSessionSearch}
/>
```

**LogSelector 功能:**
- 树形展示（按时间分组）
- 模糊搜索（使用 Fuse.js）
- 深度搜索（扫描消息内容）
- 标签过滤
- 预览面板

#### 4.4 选择交互流程

```typescript
async function onSelect(log: LogOption) {
  setResuming(true)  // 显示 "Resuming conversation..." 加载状态

  // 1. 跨项目检查
  const crossProjectCheck = checkCrossProjectResume(log, showAllProjects, worktreePaths)
  if (crossProjectCheck.isCrossProject && !crossProjectCheck.isSameRepoWorktree) {
    // 显示 cd 命令提示，复制到剪贴板
    setCrossProjectCommand(crossProjectCheck.command)
    return
  }

  // 2. 加载完整会话数据
  const result = await loadConversationForResume(log, undefined)

  // 3. 切换会话 ID
  if (result.sessionId) {
    switchSession(asSessionId(result.sessionId), dirname(log.fullPath))
    await renameRecordingForSession()
    await resetSessionFilePointer()
    restoreCostStateForSession(result.sessionId)
  }

  // 4. 恢复会话状态
  restoreSessionMetadata(result)
  restoreWorktreeForResume(result.worktreeSession)
  adoptResumedSessionFile()

  // 5. 设置恢复数据，触发 REPL 渲染
  setResumeData({
    messages: result.messages,
    fileHistorySnapshots: result.fileHistorySnapshots,
    // ...
  })
}
```

---

## 5. 恢复流程详解

### 5.1 会话数据加载

`loadConversationForResume` 是核心加载函数：

```typescript
export async function loadConversationForResume(
  source: string | LogOption | undefined,
  sourceJsonlFile: string | undefined,
): Promise<ResumeResult | null> {
  let log: LogOption | null = null
  let sessionId: UUID | undefined

  // 根据源类型加载
  if (source === undefined) {
    // --continue: 加载最近会话
    log = await loadMessageLogs()
  } else if (sourceJsonlFile) {
    // --resume with .jsonl path
    const loaded = await loadMessagesFromJsonlPath(sourceJsonlFile)
    messages = loaded.messages
  } else if (typeof source === 'string') {
    // 按 session ID 加载
    log = await getLastSessionLog(source as UUID)
  } else {
    // 已有 LogOption
    log = source
  }

  // 加载完整日志（lite log 需要额外加载）
  if (isLiteLog(log)) {
    log = await loadFullLog(log)
  }
}
```

### 5.2 状态重建

`loadTranscriptFile` 解析 JSONL 文件并重建状态：

```typescript
export async function loadTranscriptFile(filePath: string): Promise<{
  messages: Map<UUID, TranscriptMessage>
  summaries: Map<UUID, string>
  customTitles: Map<UUID, string>
  tags: Map<UUID, string>
  agentNames: Map<UUID, string>
  agentColors: Map<UUID, string>
  agentSettings: Map<UUID, string>
  modes: Map<UUID, string>
  worktreeStates: Map<UUID, PersistedWorktreeSession | null>
  fileHistorySnapshots: Map<UUID, FileHistorySnapshotMessage>
  contentReplacements: Map<UUID, ContentReplacementRecord[]>
  leafUuids: Set<UUID>  // 消息链末端
}> {
  const messages = new Map<UUID, TranscriptMessage>()
  const progressBridge = new Map<UUID, UUID | null>()  // 处理旧版进度消息

  const entries = parseJSONL<Entry>(buf)
  for (const entry of entries) {
    if (isTranscriptMessage(entry)) {
      // 桥接旧版进度消息
      if (entry.parentUuid && progressBridge.has(entry.parentUuid)) {
        entry.parentUuid = progressBridge.get(entry.parentUuid)
      }
      messages.set(entry.uuid, entry)
    } else if (entry.type === 'custom-title') {
      customTitles.set(entry.sessionId, entry.customTitle)
    } // ...处理其他元数据类型
  }

  // 计算 leafUuids（消息链末端）
  const parentUuids = new Set(allMessages.map(msg => msg.parentUuid).filter(Boolean))
  const terminalMessages = allMessages.filter(msg => !parentUuids.has(msg.uuid))
  // ...计算 leafUuids
}
```

### 5.3 消息恢复

消息通过 `buildConversationChain` 从 leaf 回溯构建：

```typescript
function buildConversationChain(
  messages: Map<UUID, TranscriptMessage>,
  leaf: TranscriptMessage
): TranscriptMessage[] {
  const chain: TranscriptMessage[] = []
  let current: TranscriptMessage | undefined = leaf

  while (current) {
    chain.unshift(current)  // 从前端插入保持时间顺序
    current = current.parentUuid
      ? messages.get(current.parentUuid)
      : undefined
  }

  return chain
}
```

### 5.4 消息反序列化

`deserializeMessagesWithInterruptDetection` 处理边缘情况：

```typescript
export function deserializeMessagesWithInterruptDetection(
  serializedMessages: Message[]
): DeserializeResult {
  // 1. 过滤未解析的工具调用
  const filteredToolUses = filterUnresolvedToolUses(migratedMessages)

  // 2. 过滤孤立的思维消息
  const filteredThinking = filterOrphanedThinkingOnlyMessages(filteredToolUses)

  // 3. 检测中断状态
  const internalState = detectTurnInterruption(filteredMessages)

  // 4. 中断的回合添加续接消息
  if (internalState.kind === 'interrupted_turn') {
    const continuationMessage = createUserMessage({
      content: 'Continue from where you left off.',
      isMeta: true
    })
    filteredMessages.push(continuationMessage)
  }

  // 5. 末尾用户消息后添加占位助手消息
  if (lastRelevantIdx !== -1 && filteredMessages[lastRelevantIdx].type === 'user') {
    filteredMessages.splice(lastRelevantIdx + 1, 0,
      createAssistantMessage({ content: NO_RESPONSE_REQUESTED })
    )
  }

  return { messages: filteredMessages, turnInterruptionState }
}
```

### 5.5 工具状态恢复

```typescript
// 文件历史恢复
if (result.fileHistorySnapshots) {
  fileHistoryRestoreStateFromLog(result.fileHistorySnapshots, newState => {
    setAppState(prev => ({ ...prev, fileHistory: newState }))
  })
}

// 内容替换记录恢复（工具结果缓存）
if (result.contentReplacements) {
  await recordContentReplacement(result.contentReplacements)
}

// Agent 状态恢复
const { agentDefinition } = restoreAgentFromSession(
  result.agentSetting,
  mainThreadAgentDefinition,
  agentDefinitions
)
setAppState(prev => ({ ...prev, agent: agentDefinition?.agentType }))
```

---

## 6. 跨项目恢复

### 6.1 项目检测逻辑

`checkCrossProjectResume` 检测是否跨项目：

```typescript
export function checkCrossProjectResume(
  log: LogOption,
  showAllProjects: boolean,
  worktreePaths: string[]
): CrossProjectResumeResult {
  const currentCwd = getOriginalCwd()

  // 未开启全局项目显示或路径相同 → 不跨项目
  if (!showAllProjects || !log.projectPath || log.projectPath === currentCwd) {
    return { isCrossProject: false }
  }

  // 检查是否同一仓库的 worktree
  const isSameRepo = worktreePaths.some(
    wt => log.projectPath === wt || log.projectPath.startsWith(wt + sep)
  )

  if (isSameRepo) {
    // 同仓库 worktree → 可直接恢复
    return { isCrossProject: true, isSameRepoWorktree: true, projectPath: log.projectPath }
  }

  // 不同仓库 → 生成 cd 命令
  const sessionId = getSessionIdFromLog(log)
  const command = `cd ${quote([log.projectPath])} && claude --resume ${sessionId}`
  return { isCrossProject: true, isSameRepoWorktree: false, command, projectPath: log.projectPath }
}
```

### 6.2 路径不匹配处理

**同仓库 worktree:**
- 直接恢复，无需切换目录
- `restoreWorktreeForResume` 会自动切换到正确的 worktree 目录

**不同仓库:**
- 显示命令提示
- 复制命令到剪贴板
- 用户需手动执行命令

```typescript
// 不同项目处理
if (crossProjectCheck.isCrossProject && !crossProjectCheck.isSameRepoWorktree) {
  const raw = await setClipboard(crossProjectCheck.command)
  if (raw) process.stdout.write(raw)  // OSC 剪贴板序列

  const message = [
    '',
    'This conversation is from a different directory.',
    '',
    'To resume, run:',
    `  ${crossProjectCheck.command}`,
    '',
    '(Command copied to clipboard)',
    ''
  ].join('\n')

  onDone(message, { display: 'user' })
}
```

### 6.3 用户体验优化

**进度式加载:**
- 首批加载 50 个会话（`INITIAL_ENRICH_COUNT`）
- 滚动加载更多，避免长等待

**lite log 优化:**
- 初始仅读取文件元数据（stat），不解析内容
- 选择会话后才加载完整内容

```typescript
// Lite log - 仅元数据
export async function getSessionFilesLite(
  projectDir: string,
  limit?: number,
  projectPath?: string
): Promise<LogOption[]> {
  // 仅 stat 文件，获取修改时间、大小等
  // 不读取文件内容
}

// 完整加载
export async function loadFullLog(log: LogOption): Promise<LogOption> {
  const { messages } = await loadTranscriptFile(sessionFile)
  log.messages = removeExtraFields(chain)
  return log
}
```

---

## 7. 用户使用指南

### 7.1 如何恢复上次会话

**方法一：使用 --continue 参数**

```bash
claude --continue
```

自动加载最近的会话（排除正在运行的后台会话）。

**方法二：使用 /resume 命令**

```bash
claude
> /resume
```

显示会话选择器，选择最近的会话。

### 7.2 如何选择历史会话

**交互式选择:**

1. 迧行 `claude` 启动 CLI
2. 入 `/resume` 命令
3. 用上下箭头键浏览会话列表
4. 按 Enter 选择会话

**会话列表显示信息:**
- 会话标题（自定义或首条消息摘要）
- 修改日期
- 消息数量
- 标签（如有）
- 项目路径（跨项目时）

**搜索功能:**
- 输入搜索词进行模糊匹配
- `/agentic` 模式可搜索消息内容

### 7.3 会话 ID 查找

**方法一：从 /status 获取**

```bash
> /status
Session ID: abc123-def456-...
```

**方法二：从文件名获取**

```bash
ls ~/.claude/projects/-project-path/
# 文件名格式：<session-id>.jsonl
```

**方法三：从 /resume 选择器**

选择器中每个会话显示其 ID。

### 7.4 恢复特定会话

**通过会话 ID:**

```bash
claude --resume abc123-def456-...
```

或在会话中：

```bash
> /resume abc123-def456-...
```

**通过自定义标题:**

```bash
> /resume my-session-name
```

需先使用 `/rename` 设置自定义标题。

### 7.5 恢复后的注意事项

**状态恢复完整性:**

- **消息历史**: 完整恢复
- **文件历史**: 通过快照恢复
- **工作目录**: 自动切换到会话退出时的目录
- **Agent 设置**: 恢复使用的 agent 类型
- **模式**: 恢复 coordinator/normal 模式

**可能丢失的状态:**

- **临时变量**: 不持久化
- **运行中的工具**: 中断的工具调用可能丢失
- **后台任务**: 需手动检查状态

**中断检测:**

如果会话在助手响应中断时退出，恢复后会自动添加：

```
Continue from where you left off.
```

提示助手继续之前的响应。

### 7.6 常见问题解决

**Q: 会话列表为空**

检查存储目录：
```bash
ls ~/.claude/projects/
```

确认会话文件存在。

**Q: 恢复失败 "Session not found"**

可能原因：
- 会话 ID 不正确
- 会话文件被删除
- 项目路径变更导致路径不匹配

解决方法：
- 检查会话文件是否存在
- 使用 `/resume` 选择器重新选择

**Q: 跨项目恢复提示需要 cd**

执行提示的命令：
```bash
cd /path/to/project && claude --resume <session-id>
```

**Q: 恢复后工作目录不对**

会话记录了退出时的目录，如果目录不存在：
- 会话会跳过目录切换
- 需手动切换到正确目录

**Q: 消息不完整**

可能原因：
- 会话在写入时崩溃
- 文件系统问题

检查文件：
```bash
cat ~/.claude/projects/-project-path/<session-id>.jsonl | head -20
```

---

## 8. 相关命令

### 8.1 /session 会话管理

显示远程会话信息（QR 码和 URL）：

```bash
> /session
```

仅在 `--remote` 模式下可用。

### 8.2 /branch 会话分支

创建当前会话的分支副本：

```bash
> /branch              # 使用默认标题
> /branch my-branch    # 使用自定义标题
```

**分支特性:**
- 复制当前会话的所有消息
- 生成新的会话 ID
- 保留原始会话的追踪信息（`forkedFrom`）
- 自动添加 "(Branch)" 后缀避免命名冲突

**实现细节:**

```typescript
async function createFork(customTitle?: string): Promise<ForkResult> {
  const forkSessionId = randomUUID()

  // 复制消息条目
  for (const entry of mainConversationEntries) {
    const forkedEntry: TranscriptEntry = {
      ...entry,
      sessionId: forkSessionId,
      forkedFrom: {
        sessionId: originalSessionId,
        messageUuid: entry.uuid
      }
    }
    lines.push(jsonStringify(forkedEntry))
  }

  // 复制内容替换记录（保持工具结果缓存）
  if (contentReplacementRecords.length > 0) {
    lines.push(jsonStringify({
      type: 'content-replacement',
      sessionId: forkSessionId,
      replacements: contentReplacementRecords
    }))
  }
}
```

### 8.3 /rewind 回退操作

（本分析未覆盖 rewind 命令，但它是会话历史操作的重要补充）

---

## 9. 从零实现指南

### 9.1 会话序列化设计

**核心数据结构:**

```typescript
interface SessionEntry {
  type: 'user' | 'assistant' | 'system' | 'attachment' | 'metadata'
  uuid: UUID
  parentUuid: UUID | null
  sessionId: UUID
  timestamp: ISOString
  content: MessageContent
}

interface SessionMetadata {
  type: 'custom-title' | 'tag' | 'mode' | 'worktree-state'
  sessionId: UUID
  value: string | object
}
```

**写入策略:**

1. **延迟物化**: 避免空会话文件
2. **批量写入**: 减少 IO 操作
3. **队列管理**: 防止并发写入冲突

```typescript
class SessionWriter {
  private queue: Entry[] = []
  private flushTimer: Timer | null = null

  write(entry: Entry): Promise<void> {
    this.queue.push(entry)
    if (!this.flushTimer) {
      this.flushTimer = setTimeout(() => this.flush(), 100)
    }
    return new Promise(resolve => this.flushResolvers.push(resolve))
  }

  private async flush(): void {
    const content = this.queue.map(e => JSON.stringify(e) + '\n').join('')
    await appendFile(this.filePath, content)
    this.queue = []
    this.flushTimer = null
    this.flushResolvers.forEach(r => r())
  }
}
```

### 9.2 状态恢复机制

**消息链重建:**

```typescript
function rebuildChain(messages: Map<UUID, Message>, leaf: UUID): Message[] {
  const chain: Message[] = []
  let current = messages.get(leaf)

  while (current) {
    chain.unshift(current)
    current = current.parentUuid ? messages.get(current.parentUuid) : null
  }

  return chain
}
```

**中断检测:**

```typescript
function detectInterruption(messages: Message[]): InterruptionState {
  const last = messages[messages.length - 1]

  if (last.type === 'assistant') {
    return { kind: 'none' }  // 正常结束
  }

  if (last.type === 'user' && !last.isMeta) {
    return { kind: 'interrupted_prompt', message: last }  // 用户已发送但未响应
  }

  if (last.type === 'user' && isToolResult(last)) {
    return { kind: 'interrupted_turn' }  // 响应中断
  }

  return { kind: 'none' }
}
```

**状态快照:**

```typescript
interface Snapshot {
  fileHistory: FileHistoryState
  attribution: AttributionState
  contentReplacements: ContentReplacement[]
  worktree: WorktreeSession | null
}

// 写入快照
async function writeSnapshot(sessionId: UUID, snapshot: Snapshot): void {
  await writeEntry({
    type: 'file-history-snapshot',
    sessionId,
    snapshot: snapshot.fileHistory
  })
  await writeEntry({
    type: 'content-replacement',
    sessionId,
    replacements: snapshot.contentReplacements
  })
}

// 恢复快照
function restoreSnapshot(entries: Entry[]): Snapshot {
  const snapshot: Snapshot = {}
  for (const entry of entries) {
    if (entry.type === 'file-history-snapshot') {
      snapshot.fileHistory = entry.snapshot
    }
    if (entry.type === 'content-replacement') {
      snapshot.contentReplacements = entry.replacements
    }
  }
  return snapshot
}
```

### 9.3 用户体验优化

**进度式加载:**

```typescript
async function loadSessionsProgressive(): Promise<SessionListResult> {
  // 1. 仅 stat 文件（不读取内容）
  const files = await readdir(sessionDir)
  const liteLogs = files.map(f => ({
    sessionId: f.name.replace('.jsonl', ''),
    modified: stat(f).mtime,
    size: stat(f).size
  }))

  // 2. 按修改时间排序
  liteLogs.sort((a, b) => b.modified - a.modified)

  // 3. 首批加载 50 个
  const enriched = await enrichLogs(liteLogs.slice(0, 50))

  return {
    logs: enriched,
    allLogs: liteLogs,
    nextIndex: 50  // 继续加载的起点
  }
}
```

**搜索增强:**

```typescript
// 模糊搜索
const fuse = new Fuse(logs, {
  keys: ['customTitle', 'firstPrompt'],
  threshold: 0.3
})

// 深度搜索（扫描消息内容）
async function deepSearch(query: string, logs: LogOption[]): Promise<LogOption[]> {
  const results: LogOption[] = []
  for (const log of logs) {
    const messages = await loadMessages(log)
    const text = messages.map(m => extractText(m)).join(' ')
    if (text.toLowerCase().includes(query.toLowerCase())) {
      results.push(log)
    }
  }
  return results
}
```

**预览面板:**

```typescript
function SessionPreview({ log }: { log: LogOption }) {
  return (
    <Box flexDirection="column">
      <Text bold>{log.customTitle || log.firstPrompt}</Text>
      <Text dimColor>{formatDate(log.modified)}</Text>
      <Text>{log.messageCount} messages</Text>
      {log.tag && <Text color="cyan">Tag: {log.tag}</Text>}
    </Box>
  )
}
```

---

## 附录：关键文件路径

| 功能 | 文件路径 |
|------|----------|
| 命令入口 | `commands/resume/index.ts` |
| 命令实现 | `commands/resume/resume.tsx` |
| 恢复屏幕 | `screens/ResumeConversation.tsx` |
| 会话存储 | `utils/sessionStorage.ts` |
| 会话恢复 | `utils/sessionRestore.ts` |
| 会话加载 | `utils/conversationRecovery.ts` |
| 跨项目检测 | `utils/crossProjectResume.ts` |
| 分支命令 | `commands/branch/branch.ts` |
| 选择器组件 | `components/LogSelector.tsx` |
| 状态管理 | `bootstrap/state.ts` |

---

*分析完成: 2026-04-02*