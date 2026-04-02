# Worktree工作树管理

**分析日期:** 2026-04-02

## 1. Worktree概述

### 1.1 什么是Worktree

Worktree是Claude Code CLI提供的一种**隔离开发环境机制**,允许用户在不影响主仓库的情况下创建独立的Git工作树进行开发。

**核心价值:**
- **隔离性**: 每个worktree拥有独立的文件系统和Git分支
- **安全性**: 主仓库不受任何修改影响
- **灵活性**: 支持中途进入/退出worktree
- **持久化**: worktree状态可保存到session,支持恢复

### 1.2 Worktree vs Git Worktree

Claude Code的Worktree基于Git worktree但进行了扩展:

| 特性 | Git Worktree | Claude Code Worktree |
|------|--------------|----------------------|
| 创建方式 | `git worktree add` | EnterWorktreeTool或`--worktree`启动参数 |
| 存储位置 | 任意目录 | `.claude/worktrees/<slug>` |
| 分支命名 | 自定义 | `worktree-<slug>` |
| Hook支持 | 无 | WorktreeCreate/WorktreeRemove hooks |
| Session集成 | 无 | 状态持久化、恢复支持 |
| tmux集成 | 无 | 可创建关联tmux session |

### 1.3 核心文件结构

```
tools/EnterWorktreeTool/
├── EnterWorktreeTool.ts    # Tool实现核心
├── constants.ts            # 工具名称常量
├── prompt.ts               # LLM提示词
└── UI.tsx                  # UI渲染组件

tools/ExitWorktreeTool/
├── ExitWorktreeTool.ts     # 退出Tool实现
├── constants.ts            # 工具名称常量
├── prompt.ts               # LLM提示词
└── UI.tsx                  # UI渲染组件

utils/
├── worktree.ts             # Worktree核心逻辑(~1100行)
├── getWorktreePaths.ts     # 获取worktree路径列表
├── sessionStorage.ts       # Worktree状态持久化
└── hooks.ts                # WorktreeCreate/Remove hooks

bootstrap/
├── state.ts                # originalCwd/projectRoot状态
└── setup.ts                # --worktree启动处理
```

---

## 2. tools/EnterWorktreeTool

### 2.1 文件定位

**路径:** `tools/EnterWorktreeTool/EnterWorktreeTool.ts`

### 2.2 工具定义

```typescript
// 第52-77行: 工具定义
export const EnterWorktreeTool: Tool<InputSchema, Output> = buildTool({
  name: ENTER_WORKTREE_TOOL_NAME,           // "EnterWorktree"
  searchHint: 'create an isolated git worktree and switch into it',
  maxResultSizeChars: 100_000,
  async description() {
    return 'Creates an isolated worktree (via git or configured hooks) and switches the session into it'
  },
  async prompt() {
    return getEnterWorktreeToolPrompt()
  },
  userFacingName() {
    return 'Creating worktree'
  },
  shouldDefer: true,                        // 需要ToolSearch才能调用
  // ...
})
```

### 2.3 输入Schema

```typescript
// 第23-39行: 输入Schema定义
const inputSchema = lazySchema(() =>
  z.strictObject({
    name: z
      .string()
      .superRefine((s, ctx) => {
        try {
          validateWorktreeSlug(s)           // 验证slug格式
        } catch (e) {
          ctx.addIssue({ code: 'custom', message: (e as Error).message })
        }
      })
      .optional()
      .describe(
        'Optional name for the worktree. Each "/"-separated segment may contain only letters, digits, dots, underscores, and dashes; max 64 chars total. A random name is generated if not provided.',
      ),
  }),
)
```

**关键点:**
- `name`参数可选,未提供时自动生成随机名称
- `validateWorktreeSlug`防止路径遍历攻击
- `/`分隔的每个segment必须符合正则`^[a-zA-Z0-9._-]+$`

### 2.4 输出Schema

```typescript
// 第42-49行: 输出Schema定义
const outputSchema = lazySchema(() =>
  z.object({
    worktreePath: z.string(),               // worktree目录路径
    worktreeBranch: z.string().optional(),  // 创建的分支名
    message: z.string(),                    // 结果消息
  }),
)
```

### 2.5 call方法逐行分析

```typescript
// 第77-119行: call方法核心实现
async call(input) {
  // === 第79-81行: 防止重复进入 ===
  if (getCurrentWorktreeSession()) {
    throw new Error('Already in a worktree session')
  }
  // 检查是否已处于worktree session中,防止嵌套

  // === 第84-88行: 定位主仓库根目录 ===
  const mainRepoRoot = findCanonicalGitRoot(getCwd())
  if (mainRepoRoot && mainRepoRoot !== getCwd()) {
    process.chdir(mainRepoRoot)
    setCwd(mainRepoRoot)
  }
  // 从当前worktree定位到主仓库,确保worktree创建在正确位置

  // === 第90行: 生成slug ===
  const slug = input.name ?? getPlanSlug()
  // 使用用户提供的名称或生成随机slug

  // === 第92行: 创建worktree ===
  const worktreeSession = await createWorktreeForSession(getSessionId(), slug)
  // 核心创建逻辑在utils/worktree.ts

  // === 第94-96行: 切换工作目录 ===
  process.chdir(worktreeSession.worktreePath)
  setCwd(worktreeSession.worktreePath)
  setOriginalCwd(getCwd())
  // 将session切换到worktree目录

  // === 第97行: 保存状态 ===
  saveWorktreeState(worktreeSession)
  // 持久化到session transcript

  // === 第99-102行: 清理缓存 ===
  clearSystemPromptSections()
  clearMemoryFileCaches()
  getPlansDirectory.cache.clear?.()
  // 清除CWD依赖的缓存,确保重新计算

  // === 第104-106行: 日志记录 ===
  logEvent('tengu_worktree_created', {
    mid_session: true,
  })

  // === 第108-118行: 返回结果 ===
  const branchInfo = worktreeSession.worktreeBranch
    ? ` on branch ${worktreeSession.worktreeBranch}`
    : ''

  return {
    data: {
      worktreePath: worktreeSession.worktreePath,
      worktreeBranch: worktreeSession.worktreeBranch,
      message: `Created worktree at ${worktreeSession.worktreePath}${branchInfo}. The session is now working in the worktree. Use ExitWorktree to leave mid-session, or exit the session to be prompted.`,
    },
  }
}
```

### 2.6 prompt.ts内容

**路径:** `tools/EnterWorktreeTool/prompt.ts`

```typescript
export function getEnterWorktreeToolPrompt(): string {
  return `Use this tool ONLY when the user explicitly asks to work in a worktree...

## When to Use
- The user explicitly says "worktree" (e.g., "start a worktree", "work in a worktree"...)

## When NOT to Use
- The user asks to create a branch, switch branches... — use git commands instead
- Never use this tool unless the user explicitly mentions "worktree"

## Requirements
- Must be in a git repository, OR have WorktreeCreate/WorktreeRemove hooks configured
- Must not already be in a worktree

## Behavior
- In a git repository: creates a new git worktree inside \`.claude/worktrees/\` with a new branch based on HEAD
- Outside a git repository: delegates to WorktreeCreate/WorktreeRemove hooks for VCS-agnostic isolation
- Switches the session's working directory to the new worktree
  `
}
```

**关键约束:** 仅在用户明确提及"worktree"时使用

### 2.7 UI.tsx渲染

**路径:** `tools/EnterWorktreeTool/UI.tsx`

```typescript
export function renderToolUseMessage(): React.ReactNode {
  return 'Creating worktree…';
}

export function renderToolResultMessage(output: Output, ...): React.ReactNode {
  return <Box flexDirection="column">
      <Text>
        Switched to worktree on branch <Text bold>{output.worktreeBranch}</Text>
      </Text>
      <Text dimColor>{output.worktreePath}</Text>
    </Box>;
}
```

---

## 3. tools/ExitWorktreeTool

### 3.1 文件定位

**路径:** `tools/ExitWorktreeTool/ExitWorktreeTool.ts`

### 3.2 工具定义

```typescript
// 第148-174行: 工具定义
export const ExitWorktreeTool: Tool<InputSchema, Output> = buildTool({
  name: EXIT_WORKTREE_TOOL_NAME,            // "ExitWorktree"
  searchHint: 'exit a worktree session and return to the original directory',
  maxResultSizeChars: 100_000,
  async description() {
    return 'Exits a worktree session created by EnterWorktree and restores the original working directory'
  },
  userFacingName() {
    return 'Exiting worktree'
  },
  shouldDefer: true,
  isDestructive(input) {
    return input.action === 'remove'        // remove操作标记为破坏性
  },
  toAutoClassifierInput(input) {
    return input.action
  },
  // ...
})
```

### 3.3 输入Schema

```typescript
// 第30-44行: 输入Schema
const inputSchema = lazySchema(() =>
  z.strictObject({
    action: z
      .enum(['keep', 'remove'])
      .describe('"keep" leaves the worktree and branch on disk; "remove" deletes both.'),
    discard_changes: z
      .boolean()
      .optional()
      .describe('Required true when action is "remove" and the worktree has uncommitted files...'),
  }),
)
```

**action参数:**
- `keep`: 保留worktree目录和分支,仅切换回原目录
- `remove`: 删除worktree目录和分支

**discard_changes参数:**
- 当worktree有未提交文件或未合并commit时必须设为true
- 作为安全确认机制

### 3.4 输出Schema

```typescript
// 第47-60行: 输出Schema
const outputSchema = lazySchema(() =>
  z.object({
    action: z.enum(['keep', 'remove']),
    originalCwd: z.string(),                // 原始工作目录
    worktreePath: z.string(),               // worktree路径
    worktreeBranch: z.string().optional(),  // 分支名
    tmuxSessionName: z.string().optional(), // tmux session名
    discardedFiles: z.number().optional(),  // 丢弃的文件数
    discardedCommits: z.number().optional(),// 丢弃的commit数
    message: z.string(),
  }),
)
```

### 3.5 validateInput方法

```typescript
// 第174-224行: 输入验证
async validateInput(input) {
  // === 第180-188行: Scope guard ===
  const session = getCurrentWorktreeSession()
  if (!session) {
    return {
      result: false,
      message: 'No-op: there is no active EnterWorktree session to exit...',
      errorCode: 1,
    }
  }
  // 确保只在EnterWorktree创建的session中操作

  // === 第190-220行: remove操作的安全检查 ===
  if (input.action === 'remove' && !input.discard_changes) {
    const summary = await countWorktreeChanges(
      session.worktreePath,
      session.originalHeadCommit,
    )
    if (summary === null) {
      // 无法验证状态,拒绝操作
      return { result: false, message: 'Could not verify worktree state...', errorCode: 3 }
    }
    const { changedFiles, commits } = summary
    if (changedFiles > 0 || commits > 0) {
      // 有未提交的修改,需要用户确认
      return {
        result: false,
        message: `Worktree has ${changedFiles} uncommitted files and ${commits} commits...`,
        errorCode: 2,
      }
    }
  }

  return { result: true }
}
```

### 3.6 countWorktreeChanges函数

```typescript
// 第79-113行: 统计worktree变更
async function countWorktreeChanges(
  worktreePath: string,
  originalHeadCommit: string | undefined,
): Promise<ChangeSummary | null> {
  // === 第83-92行: 检查未提交文件 ===
  const status = await execFileNoThrow('git', [
    '-C', worktreePath,
    'status', '--porcelain',
  ])
  if (status.code !== 0) {
    return null  // fail-closed: git失败时返回null
  }
  const changedFiles = count(status.stdout.split('\n'), l => l.trim() !== '')

  // === 第94-98行: 无baseline时返回null ===
  if (!originalHeadCommit) {
    return null  // hook-based worktree可能没有originalHeadCommit
  }

  // === 第100-110行: 统计commit数 ===
  const revList = await execFileNoThrow('git', [
    '-C', worktreePath,
    'rev-list', '--count',
    `${originalHeadCommit}..HEAD`,
  ])
  if (revList.code !== 0) {
    return null
  }
  const commits = parseInt(revList.stdout.trim(), 10) || 0

  return { changedFiles, commits }
}
```

### 3.7 restoreSessionToOriginalCwd函数

```typescript
// 第122-146行: 恢复session到原目录
function restoreSessionToOriginalCwd(
  originalCwd: string,
  projectRootIsWorktree: boolean,
): void {
  setCwd(originalCwd)
  setOriginalCwd(originalCwd)
  
  // === 第135-141行: 仅在projectRoot被设置时恢复 ===
  if (projectRootIsWorktree) {
    setProjectRoot(originalCwd)
    updateHooksConfigSnapshot()  // 重新读取hooks配置
  }
  
  saveWorktreeState(null)        // 清除worktree状态
  clearSystemPromptSections()
  clearMemoryFileCaches()
  getPlansDirectory.cache.clear?.()
}
```

**关键注释(第130-134行):**
```typescript
// --worktree startup sets projectRoot to the worktree; mid-session
// EnterWorktreeTool does not. Only restore when it was actually changed —
// otherwise we'd move projectRoot to wherever the user had cd'd before
// entering the worktree, breaking the "stable project identity" contract.
```

### 3.8 call方法核心流程

```typescript
// 第227-321行: call方法
async call(input) {
  const session = getCurrentWorktreeSession()
  if (!session) {
    throw new Error('Not in a worktree session')
  }

  // 提取session信息
  const { originalCwd, worktreePath, worktreeBranch, tmuxSessionName, originalHeadCommit } = session

  // 判断projectRoot是否是worktree
  const projectRootIsWorktree = getProjectRoot() === getOriginalCwd()

  // 统计变更(用于日志和输出)
  const { changedFiles, commits } = (await countWorktreeChanges(worktreePath, originalHeadCommit))
    ?? { changedFiles: 0, commits: 0 }

  if (input.action === 'keep') {
    // === keep操作 ===
    await keepWorktree()                      // utils/worktree.ts
    restoreSessionToOriginalCwd(originalCwd, projectRootIsWorktree)
    
    logEvent('tengu_worktree_kept', { mid_session: true, commits, changed_files: changedFiles })
    
    const tmuxNote = tmuxSessionName
      ? ` Tmux session ${tmuxSessionName} is still running; reattach with: tmux attach -t ${tmuxSessionName}`
      : ''
    
    return { data: { action: 'keep', originalCwd, worktreePath, worktreeBranch, tmuxSessionName, message: ... } }
  }

  // === remove操作 ===
  if (tmuxSessionName) {
    await killTmuxSession(tmuxSessionName)
  }
  await cleanupWorktree()                    // utils/worktree.ts
  restoreSessionToOriginalCwd(originalCwd, projectRootIsWorktree)
  
  logEvent('tengu_worktree_removed', { mid_session: true, commits, changed_files: changedFiles })
  
  return { data: { action: 'remove', ... } }
}
```

### 3.9 UI.tsx渲染

```typescript
export function renderToolUseMessage(): React.ReactNode {
  return 'Exiting worktree…';
}

export function renderToolResultMessage(output: Output, ...): React.ReactNode {
  const actionLabel = output.action === 'keep' ? 'Kept worktree' : 'Removed worktree';
  return <Box flexDirection="column">
      <Text>
        {actionLabel}
        {output.worktreeBranch ? <>
            {' '} (branch <Text bold>{output.worktreeBranch}</Text>)
          </> : null}
      </Text>
      <Text dimColor>Returned to {output.originalCwd}</Text>
    </Box>;
}
```

---

## 4. 隔离机制

### 4.1 utils/worktree.ts核心类型

```typescript
// 第140-154行: WorktreeSession类型定义
export type WorktreeSession = {
  originalCwd: string           // 进入前的原始目录
  worktreePath: string          // worktree目录路径
  worktreeName: string          // worktree名称(slug)
  worktreeBranch?: string       // 创建的分支名
  originalBranch?: string       // 原始分支名
  originalHeadCommit?: string   // 原始HEAD commit SHA
  sessionId: string             // 关联的session ID
  tmuxSessionName?: string      // tmux session名(可选)
  hookBased?: boolean           // 是否基于hook创建
  creationDurationMs?: number   // 创建耗时
  usedSparsePaths?: boolean     // 是否使用了sparse-checkout
}
```

### 4.2 全局session状态

```typescript
// 第156-160行: 模块级状态
let currentWorktreeSession: WorktreeSession | null = null

export function getCurrentWorktreeSession(): WorktreeSession | null {
  return currentWorktreeSession
}

export function restoreWorktreeSession(session: WorktreeSession | null): void {
  currentWorktreeSession = session
}
```

### 4.3 Slug验证(安全机制)

```typescript
// 第48-87行: Slug验证防止路径遍历
const VALID_WORKTREE_SLUG_SEGMENT = /^[a-zA-Z0-9._-]+$/
const MAX_WORKTREE_SLUG_LENGTH = 64

export function validateWorktreeSlug(slug: string): void {
  if (slug.length > MAX_WORKTREE_SLUG_LENGTH) {
    throw new Error(`Invalid worktree name: must be ${MAX_WORKTREE_SLUG_LENGTH} characters or fewer...`)
  }
  
  for (const segment of slug.split('/')) {
    if (segment === '.' || segment === '..') {
      throw new Error(`Invalid worktree name "${slug}": must not contain "." or ".." path segments`)
    }
    if (!VALID_WORKTREE_SLUG_SEGMENT.test(segment)) {
      throw new Error(`Invalid worktree name "${slug}": each "/"-separated segment must...`)
    }
  }
}
```

**安全考虑:**
- `path.join`会规范化`..`段,可能导致路径逃逸
- 绝对路径(`/`或`C:\`)会丢弃前缀
- `/`分隔允许嵌套命名如`user/feature`,但每段独立验证

### 4.4 路径与分支命名

```typescript
// 第204-228行: 路径和分支生成
function worktreesDir(repoRoot: string): string {
  return join(repoRoot, '.claude', 'worktrees')
}

// Flatten nested slugs (`user/feature` → `user+feature`)
// `+` is valid in git branch names but NOT in slug-segment allowlist
function flattenSlug(slug: string): string {
  return slug.replaceAll('/', '+')
}

export function worktreeBranchName(slug: string): string {
  return `worktree-${flattenSlug(slug)}`
}

function worktreePathFor(repoRoot: string, slug: string): string {
  return join(worktreesDir(repoRoot), flattenSlug(slug))
}
```

**为什么flatten:**
- Git refs: `worktree-user`(文件) vs `worktree-user/feature`(需要目录)会D/F冲突
- 目录: `.claude/worktrees/user/feature/`嵌套在`user`内,父级删除会影响子级
- `+`在branch name合法但不在slug allowlist中,保证映射可逆

### 4.5 getOrCreateWorktree

```typescript
// 第235-375行: 创建或恢复worktree
async function getOrCreateWorktree(
  repoRoot: string,
  slug: string,
  options?: { prNumber?: number },
): Promise<WorktreeCreateResult> {
  const worktreePath = worktreePathFor(repoRoot, slug)
  const worktreeBranch = worktreeBranchName(slug)

  // === 第247-255行: 快速恢复路径 ===
  const existingHead = await readWorktreeHeadSha(worktreePath)
  if (existingHead) {
    return {
      worktreePath,
      worktreeBranch,
      headCommit: existingHead,
      existed: true,             // 标记为已存在
    }
  }
  // 直接读取.git pointer文件,避免spawn subprocess

  // === 第258-303行: 获取base branch ===
  await mkdir(worktreesDir(repoRoot), { recursive: true })
  
  let baseBranch: string
  let baseSha: string | null = null
  
  if (options?.prNumber) {
    // PR模式: fetch PR head
    await execFileNoThrowWithCwd(gitExe(), ['fetch', 'origin', `pull/${options.prNumber}/head`], ...)
    baseBranch = 'FETCH_HEAD'
  } else {
    // 正常模式: 检查origin/<branch>是否已存在
    const originSha = await resolveRef(gitDir, `refs/remotes/origin/${defaultBranch}`)
    if (originSha) {
      baseBranch = originRef
      baseSha = originSha         // 跳过fetch
    } else {
      await execFileNoThrowWithCwd(gitExe(), ['fetch', 'origin', defaultBranch], ...)
      baseBranch = originRef
    }
  }

  // === 第321-334行: 创建worktree ===
  const addArgs = ['worktree', 'add']
  if (sparsePaths?.length) {
    addArgs.push('--no-checkout')
  }
  addArgs.push('-B', worktreeBranch, worktreePath, baseBranch)
  // -B (not -b): reset orphan branch,节省git branch -D subprocess

  await execFileNoThrowWithCwd(gitExe(), addArgs, { cwd: repoRoot })

  // === 第336-366行: sparse-checkout处理 ===
  if (sparsePaths?.length) {
    await execFileNoThrowWithCwd(gitExe(), ['sparse-checkout', 'set', '--cone', '--', ...sparsePaths], ...)
    await execFileNoThrowWithCwd(gitExe(), ['checkout', 'HEAD'], ...)
  }

  return { worktreePath, worktreeBranch, headCommit: baseSha, baseBranch, existed: false }
}
```

### 4.6 performPostCreationSetup

```typescript
// 第510-624行: 创建后设置
async function performPostCreationSetup(repoRoot: string, worktreePath: string): Promise<void> {
  // === 第516-534行: 复制settings.local.json ===
  const sourceSettingsLocal = join(repoRoot, localSettingsRelativePath)
  const destSettingsLocal = join(worktreePath, localSettingsRelativePath)
  await mkdirRecursive(dirname(destSettingsLocal))
  await copyFile(sourceSettingsLocal, destSettingsLocal)

  // === 第536-578行: 配置git hooks路径 ===
  const huskyPath = join(repoRoot, '.husky')
  const gitHooksPath = join(repoRoot, '.git', 'hooks')
  // 设置core.hooksPath指向主仓库,解决相对路径问题

  // === 第580-585行: symlink目录 ===
  const dirsToSymlink = settings.worktree?.symlinkDirectories ?? []
  if (dirsToSymlink.length > 0) {
    await symlinkDirectories(repoRoot, worktreePath, dirsToSymlink)
  }

  // === 第587-588行: 复制.worktreeinclude文件 ===
  await copyWorktreeIncludeFiles(repoRoot, worktreePath)

  // === 第599-623行: 安装attribution hook ===
  if (feature('COMMIT_ATTRIBUTION')) {
    await import('./postCommitAttribution.js').then(m => m.installPrepareCommitMsgHook(...))
  }
}
```

### 4.7 createWorktreeForSession

```typescript
// 第702-778行: Session级创建入口
export async function createWorktreeForSession(
  sessionId: string,
  slug: string,
  tmuxSessionName?: string,
  options?: { prNumber?: number },
): Promise<WorktreeSession> {
  validateWorktreeSlug(slug)               // 安全验证

  const originalCwd = getCwd()

  // === 第715-728行: Hook-based路径 ===
  if (hasWorktreeCreateHook()) {
    const hookResult = await executeWorktreeCreateHook(slug)
    currentWorktreeSession = {
      originalCwd,
      worktreePath: hookResult.worktreePath,
      worktreeName: slug,
      sessionId,
      tmuxSessionName,
      hookBased: true,
    }
  } else {
    // === 第730-768行: Git-based路径 ===
    const gitRoot = findGitRoot(getCwd())
    if (!gitRoot) {
      throw new Error('Cannot create a worktree: not in a git repository...')
    }

    const { worktreePath, worktreeBranch, headCommit, existed } =
      await getOrCreateWorktree(gitRoot, slug, options)

    if (!existed) {
      await performPostCreationSetup(gitRoot, worktreePath)
    }

    currentWorktreeSession = {
      originalCwd,
      worktreePath,
      worktreeName: slug,
      worktreeBranch,
      originalBranch: await getBranch(),
      originalHeadCommit: headCommit,
      sessionId,
      tmuxSessionName,
      // ...
    }
  }

  // === 第771-777行: 持久化到project config ===
  saveCurrentProjectConfig(current => ({
    ...current,
    activeWorktreeSession: currentWorktreeSession ?? undefined,
  }))

  return currentWorktreeSession
}
```

### 4.8 keepWorktree与cleanupWorktree

```typescript
// 第780-811行: keepWorktree
export async function keepWorktree(): Promise<void> {
  if (!currentWorktreeSession) return

  const { worktreePath, originalCwd, worktreeBranch } = currentWorktreeSession

  process.chdir(originalCwd)               // 先切换回原目录
  currentWorktreeSession = null            // 清除session状态

  saveCurrentProjectConfig(current => ({
    ...current,
    activeWorktreeSession: undefined,
  }))
}

// 第813-894行: cleanupWorktree
export async function cleanupWorktree(): Promise<void> {
  if (!currentWorktreeSession) return

  const { worktreePath, originalCwd, worktreeBranch, hookBased } = currentWorktreeSession

  process.chdir(originalCwd)

  if (hookBased) {
    // Hook-based: 委托给WorktreeRemove hook
    await executeWorktreeRemoveHook(worktreePath)
  } else {
    // Git-based: git worktree remove --force
    await execFileNoThrowWithCwd(gitExe(), ['worktree', 'remove', '--force', worktreePath], ...)
    
    // 删除临时分支
    await sleep(100)                       // 等待git释放锁
    await execFileNoThrowWithCwd(gitExe(), ['branch', '-D', worktreeBranch], ...)
  }

  currentWorktreeSession = null
  saveCurrentProjectConfig(current => ({ ...current, activeWorktreeSession: undefined }))
}
```

### 4.9 Hook机制

**utils/hooks.ts相关函数:**

```typescript
// 第4910-4920行: 检查是否有WorktreeCreate hook
export function hasWorktreeCreateHook(): boolean {
  const snapshotHooks = getHooksConfigFromSnapshot()?.['WorktreeCreate']
  if (snapshotHooks && snapshotHooks.length > 0) return true
  const registeredHooks = getRegisteredHooks()?.['WorktreeCreate']
  if (!registeredHooks || registeredHooks.length === 0) return false
  // Mirror getHooksConfig(): skip plugin hooks in managed-only mode
  const managedOnly = shouldAllowManagedHooksOnly()
  // ...
}

// 第4928-4958行: 执行WorktreeCreate hook
export async function executeWorktreeCreateHook(name: string): Promise<{ worktreePath: string }> {
  const hookInput = {
    ...createBaseHookInput(undefined),
    hook_event_name: 'WorktreeCreate' as const,
    name,
  }
  // 执行并获取worktreePath输出
}

// 第4967行: 执行WorktreeRemove hook
export async function executeWorktreeRemoveHook(worktreePath: string): Promise<boolean> {
  const hookInput = {
    ...createBaseHookInput(undefined),
    hook_event_name: 'WorktreeRemove' as const,
    worktree_path: worktreePath,
  }
  // ...
}
```

**Hook支持使得worktree可以用于非Git VCS系统(如SVN、P4等).**

### 4.10 状态持久化

**utils/sessionStorage.ts:**

```typescript
// 第2889-2920行: 保存worktree状态
export function saveWorktreeState(worktreeSession: PersistedWorktreeSession | null): void {
  // Strip ephemeral fields (creationDurationMs, usedSparsePaths)
  const stripped: PersistedWorktreeSession | null = worktreeSession
    ? {
        originalCwd: worktreeSession.originalCwd,
        worktreePath: worktreeSession.worktreePath,
        worktreeName: worktreeSession.worktreeName,
        worktreeBranch: worktreeSession.worktreeBranch,
        originalBranch: worktreeSession.originalBranch,
        sessionId: worktreeSession.sessionId,
        // 不包含creationDurationMs, usedSparsePaths
      }
    : null

  const project = getProject()
  project.currentSessionWorktree = stripped

  // Write eagerly when the file already exists (mid-session enter/exit)
  if (project.sessionFile) {
    appendEntryToFile(project.sessionFile, {
      type: 'worktree-state',
      worktreeSession: stripped,
      uuid: randomUUID(),
      timestamp: new Date().toISOString(),
    })
  }
}
```

---

## 5. 用户使用指南

### 5.1 启动时创建Worktree

使用`--worktree`启动参数:

```bash
# 自动创建worktree并切换
claude --worktree

# 指定worktree名称
claude --worktree --worktree-name my-feature

# 创建tmux session关联
claude --worktree --tmux

# 从PR创建worktree
claude --worktree --pr 123
```

### 5.2 中途进入Worktree

在对话中请求:

```
用户: "请在一个worktree中实现这个功能"
Claude: [调用EnterWorktreeTool]
→ Created worktree at .claude/worktrees/worktree-abc123 on branch worktree-abc123
```

### 5.3 中途退出Worktree

```
用户: "退出worktree并保留我的修改"
Claude: [调用ExitWorktreeTool with action: "keep"]
→ Kept worktree (branch worktree-abc123)
→ Returned to /path/to/original/repo

用户: "退出并删除worktree"
Claude: [调用ExitWorktreeTool with action: "remove"]
→ Removed worktree
→ Returned to /path/to/original/repo
```

### 5.4 恢复Session时的Worktree

当使用`--resume`恢复session时:

```typescript
// setup.ts处理
if (worktreeSession) {
  process.chdir(worktreeSession.worktreePath)
  setCwd(worktreeSession.worktreePath)
  setOriginalCwd(getCwd())
  setProjectRoot(getCwd())           // worktree作为project root
  saveWorktreeState(worktreeSession)
}
```

### 5.5 配置选项

**settings.json中的worktree配置:**

```json
{
  "worktree": {
    "sparsePaths": ["src/", "tests/"],           // sparse-checkout路径
    "symlinkDirectories": ["node_modules/"]      // symlink目录避免重复
  }
}
```

**.worktreeinclude文件:**

指定需要复制到worktree的gitignored文件:

```
# .worktreeinclude示例
.env.local
config/secrets/
*.key
```

### 5.6 Worktree清理机制

**EPHEMERAL_WORKTREE_PATTERNS(第1030-1041行):**

```typescript
const EPHEMERAL_WORKTREE_PATTERNS = [
  /^agent-a[0-9a-f]{7}$/,                       // AgentTool创建
  /^wf_[0-9a-f]{8}-[0-9a-f]{3}-\d+$/,           // WorkflowTool创建
  /^wf-\d+$/,                                   // Legacy workflow
  /^bridge-[A-Za-z0-9_]+(-[A-Za-z0-9_]+)*$/,    // Bridge session
  /^job-[a-zA-Z0-9._-]{1,55}-[0-9a-f]{8}$/,     // Template job
]
```

**cleanupStaleAgentWorktrees(第1058-1098行):**
- 清理30天以上的ephemeral worktrees
- 不清理用户命名的EnterWorktree worktrees
- fail-closed: 状态无法验证时跳过

---

## 6. 从零实现

### 6.1 最小化实现

```typescript
// === 核心类型定义 ===
type WorktreeSession = {
  originalCwd: string
  worktreePath: string
  worktreeBranch: string
}

let currentWorktreeSession: WorktreeSession | null = null

// === 进入Worktree ===
async function enterWorktree(slug: string): Promise<WorktreeSession> {
  if (currentWorktreeSession) {
    throw new Error('Already in a worktree')
  }

  const originalCwd = process.cwd()
  const gitRoot = findGitRoot(originalCwd)
  if (!gitRoot) throw new Error('Not in a git repository')

  // 创建路径和分支
  const worktreePath = join(gitRoot, '.claude', 'worktrees', slug)
  const worktreeBranch = `worktree-${slug}`

  // Git worktree add
  await execFile('git', ['worktree', 'add', '-b', worktreeBranch, worktreePath, 'HEAD'])

  // 切换目录
  process.chdir(worktreePath)

  currentWorktreeSession = {
    originalCwd,
    worktreePath,
    worktreeBranch,
  }

  return currentWorktreeSession
}

// === 退出Worktree ===
async function exitWorktree(action: 'keep' | 'remove'): Promise<void> {
  if (!currentWorktreeSession) {
    throw new Error('Not in a worktree')
  }

  const { originalCwd, worktreePath, worktreeBranch } = currentWorktreeSession

  process.chdir(originalCwd)

  if (action === 'remove') {
    await execFile('git', ['worktree', 'remove', '--force', worktreePath])
    await execFile('git', ['branch', '-D', worktreeBranch])
  }

  currentWorktreeSession = null
}
```

### 6.2 完整Tool集成

```typescript
import { buildTool } from './Tool.js'
import { z } from 'zod'

const enterWorktreeSchema = z.object({
  name: z.string().optional().describe('Worktree name'),
})

export const EnterWorktreeTool = buildTool({
  name: 'EnterWorktree',
  description: async () => 'Create an isolated worktree and switch into it',
  inputSchema: enterWorktreeSchema,
  shouldDefer: true,

  async call(input) {
    const slug = input.name ?? generateSlug()
    const session = await enterWorktree(slug)

    return {
      data: {
        worktreePath: session.worktreePath,
        worktreeBranch: session.worktreeBranch,
        message: `Created worktree at ${session.worktreePath}`,
      },
    }
  },

  renderToolUseMessage: () => 'Creating worktree…',
  renderToolResultMessage: (output) => `Switched to ${output.worktreePath}`,
})

const exitWorktreeSchema = z.object({
  action: z.enum(['keep', 'remove']),
  discard_changes: z.boolean().optional(),
})

export const ExitWorktreeTool = buildTool({
  name: 'ExitWorktree',
  description: async () => 'Exit worktree and return to original directory',
  inputSchema: exitWorktreeSchema,
  shouldDeer: true,
  isDestructive: (input) => input.action === 'remove',

  async validateInput(input) {
    if (!currentWorktreeSession) {
      return { result: false, message: 'No active worktree session' }
    }

    if (input.action === 'remove' && !input.discard_changes) {
      const hasChanges = await checkUncommittedChanges(currentWorktreeSession.worktreePath)
      if (hasChanges) {
        return { result: false, message: 'Has uncommitted changes, set discard_changes: true' }
      }
    }

    return { result: true }
  },

  async call(input) {
    await exitWorktree(input.action)
    return { data: { message: `Exited worktree with action: ${input.action}` } }
  },
})
```

### 6.3 Hook扩展实现

```typescript
// === Hook类型定义 ===
type WorktreeCreateHookInput = {
  hook_event_name: 'WorktreeCreate'
  name: string
}

type WorktreeRemoveHookInput = {
  hook_event_name: 'WorktreeRemove'
  worktree_path: string
}

// === Hook执行 ===
async function executeWorktreeCreateHook(name: string): Promise<string> {
  const hooks = getHooksForEvent('WorktreeCreate')
  if (hooks.length === 0) throw new Error('No WorktreeCreate hooks configured')

  const input: WorktreeCreateHookInput = { hook_event_name: 'WorktreeCreate', name }

  for (const hook of hooks) {
    const result = await executeHook(hook, input)
    if (result.worktreePath) {
      return result.worktreePath
    }
  }

  throw new Error('WorktreeCreate hook did not return worktreePath')
}

// === 创建逻辑整合 ===
async function createWorktreeForSession(slug: string): Promise<WorktreeSession> {
  const hasHook = hasWorktreeCreateHook()
  const inGit = await getIsGit()

  if (!hasHook && !inGit) {
    throw new Error('Need git repo or WorktreeCreate hook')
  }

  if (hasHook) {
    const worktreePath = await executeWorktreeCreateHook(slug)
    return { originalCwd: getCwd(), worktreePath, worktreeBranch: undefined, hookBased: true }
  }

  // Git-based creation...
}
```

### 6.4 状态持久化实现

```typescript
// === Session transcript记录 ===
type WorktreeStateEntry = {
  type: 'worktree-state'
  worktreeSession: {
    originalCwd: string
    worktreePath: string
    worktreeBranch?: string
  }
  timestamp: string
}

// === 保存状态 ===
function saveWorktreeState(session: WorktreeSession | null): void {
  const entry: WorktreeStateEntry = {
    type: 'worktree-state',
    worktreeSession: session ? {
      originalCwd: session.originalCwd,
      worktreePath: session.worktreePath,
      worktreeBranch: session.worktreeBranch,
    } : null,
    timestamp: new Date().toISOString(),
  }

  appendToTranscript(entry)
}

// === 恢复状态 ===
function loadWorktreeState(transcript: Entry[]): WorktreeSession | null {
  const lastWorktreeEntry = transcript
    .filter(e => e.type === 'worktree-state')
    .pop()

  return lastWorktreeEntry?.worktreeSession ?? null
}
```

### 6.5 安全考虑实现要点

```typescript
// === Slug验证 ===
function validateSlug(slug: string): void {
  // 长度限制
  if (slug.length > 64) throw new Error('Name too long')

  // 路径遍历防护
  for (const segment of slug.split('/')) {
    if (segment === '.' || segment === '..') throw new Error('Invalid segment')
    if (!/^[a-zA-Z0-9._-]+$/.test(segment)) throw new Error('Invalid characters')
  }
}

// === 变更检查 ===
async function checkUncommittedChanges(worktreePath: string): Promise<boolean> {
  const { stdout } = await execFile('git', ['-C', worktreePath, 'status', '--porcelain'])
  return stdout.trim().length > 0
}

// === 删除前确认 ===
async function safeRemoveWorktree(worktreePath: string): Promise<void> {
  const hasChanges = await checkUncommittedChanges(worktreePath)
  if (hasChanges) {
    throw new Error('Worktree has uncommitted changes, cannot remove')
  }

  await execFile('git', ['worktree', 'remove', worktreePath])
}
```

---

## 附录: 关键文件路径索引

| 功能 | 文件路径 |
|------|----------|
| EnterWorktreeTool定义 | `tools/EnterWorktreeTool/EnterWorktreeTool.ts` |
| ExitWorktreeTool定义 | `tools/ExitWorktreeTool/ExitWorktreeTool.ts` |
| Worktree核心逻辑 | `utils/worktree.ts` |
| Hook执行 | `utils/hooks.ts` |
| 状态持久化 | `utils/sessionStorage.ts` |
| Bootstrap状态 | `bootstrap/state.ts` |
| 启动处理 | `setup.ts` |
| Worktree路径获取 | `utils/getWorktreePaths.ts` |

---

*Worktree分析完成: 2026-04-02*