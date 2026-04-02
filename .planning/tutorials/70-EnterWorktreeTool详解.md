# EnterWorktreeTool 详解

**分析日期:** 2026-04-02

## 1. 工具概述

### 1.1 功能定义

`EnterWorktreeTool` 是 Claude Code CLI 中用于创建隔离 Git Worktree 环境的核心工具。它允许用户在会话中创建一个独立的工作树，与主仓库分离，实现以下目标：

- **隔离开发环境**：在不影响主分支的情况下进行实验性开发
- **并行任务处理**：同时在不同分支上处理多个任务
- **安全测试**：在独立环境中测试危险的修改操作

### 1.2 核心文件结构

```
tools/EnterWorktreeTool/
├── EnterWorktreeTool.ts    # 主工具定义与执行逻辑
├── constants.ts            # 工具名称常量
├── prompt.ts               # 工具使用提示词
├── UI.tsx                  # 用户界面渲染组件
```

### 1.3 关键依赖

| 模块 | 文件路径 | 功能 |
|------|----------|------|
| `createWorktreeForSession` | `utils/worktree.ts` | Worktree 创建核心逻辑 |
| `validateWorktreeSlug` | `utils/worktree.ts` | 名称安全验证 |
| `findCanonicalGitRoot` | `utils/git.ts` | Git 根目录定位 |
| `saveWorktreeState` | `utils/sessionStorage.ts` | 状态持久化 |
| `getSessionId` | `bootstrap/state.ts` | 会话 ID 获取 |
| `clearSystemPromptSections` | `constants/systemPromptSections.ts` | 清除系统提示缓存 |

---

## 2. 输入 Schema

### 2.1 Schema 定义（逐行分析）

```typescript
// 文件：tools/EnterWorktreeTool/EnterWorktreeTool.ts (第 23-39 行)

const inputSchema = lazySchema(() =>
  z.strictObject({
    name: z
      .string()
      .superRefine((s, ctx) => {
        try {
          validateWorktreeSlug(s)
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

**逐行解析：**

| 行号 | 代码 | 解析 |
|------|------|------|
| 23 | `lazySchema(() => ...)` | 延迟初始化 Schema，避免模块加载时的 Zod 构造开销 |
| 24 | `z.strictObject({...})` | 严格对象模式，禁止未知字段 |
| 25-27 | `name: z.string().superRefine(...)` | 使用 superRefine 进行自定义验证 |
| 28-32 | `validateWorktreeSlug(s)` | 调用安全验证函数，防止路径遍历攻击 |
| 33 | `ctx.addIssue(...)` | 验证失败时添加错误信息 |
| 34 | `.optional()` | 名称可选，不提供时自动生成随机名称 |
| 35-37 | `.describe(...)` | 提供工具使用说明 |

### 2.2 名称验证规则

`validateWorktreeSlug` 函数（`utils/worktree.ts` 第 66-87 行）实现以下安全验证：

```typescript
const VALID_WORKTREE_SLUG_SEGMENT = /^[a-zA-Z0-9._-]+$/
const MAX_WORKTREE_SLUG_LENGTH = 64

export function validateWorktreeSlug(slug: string): void {
  // 1. 长度检查：最大 64 字符
  if (slug.length > MAX_WORKTREE_SLUG_LENGTH) {
    throw new Error(`Invalid worktree name: must be ${MAX_WORKTREE_SLUG_LENGTH} characters or fewer`)
  }
  
  // 2. 路径遍历防护：禁止 "." 和 ".." 段
  for (const segment of slug.split('/')) {
    if (segment === '.' || segment === '..') {
      throw new Error(`Invalid worktree name: must not contain "." or ".." path segments`)
    }
    
    // 3. 字符白名单：只允许字母、数字、点、下划线、横线
    if (!VALID_WORKTREE_SLUG_SEGMENT.test(segment)) {
      throw new Error(`Invalid worktree name: each segment must contain only letters, digits, dots, underscores, and dashes`)
    }
  }
}
```

**安全设计要点：**

| 检查项 | 目的 | 防护对象 |
|--------|------|----------|
| 长度限制 | 防止超长路径名 | 文件系统兼容性 |
| 路径遍历 | 防止 `../../../etc/passwd` | 目录逃逸攻击 |
| 字符白名单 | 防止特殊字符注入 | Shell 命令注入、Git 分支名冲突 |

---

## 3. Worktree 创建

### 3.1 主执行流程（逐行分析）

```typescript
// 文件：tools/EnterWorktreeTool/EnterWorktreeTool.ts (第 77-119 行)

async call(input) {
  // [第 78-81 行] 验证不已在 Worktree 会话中
  if (getCurrentWorktreeSession()) {
    throw new Error('Already in a worktree session')
  }

  // [第 83-88 行] 切换到主仓库根目录
  const mainRepoRoot = findCanonicalGitRoot(getCwd())
  if (mainRepoRoot && mainRepoRoot !== getCwd()) {
    process.chdir(mainRepoRoot)
    setCwd(mainRepoRoot)
  }

  // [第 90 行] 获取或生成 Worktree 名称
  const slug = input.name ?? getPlanSlug()

  // [第 92 行] 创建 Worktree 会话
  const worktreeSession = await createWorktreeForSession(getSessionId(), slug)

  // [第 94-96 行] 切换工作目录到 Worktree
  process.chdir(worktreeSession.worktreePath)
  setCwd(worktreeSession.worktreePath)
  setOriginalCwd(getCwd())
  
  // [第 97 行] 持久化状态
  saveWorktreeState(worktreeSession)
  
  // [第 99-102 行] 清除缓存以刷新上下文
  clearSystemPromptSections()
  clearMemoryFileCaches()
  getPlansDirectory.cache.clear?.()

  // [第 104-106 行] 记录分析事件
  logEvent('tengu_worktree_created', { mid_session: true })

  // [第 108-118 行] 返回结果
  return {
    data: {
      worktreePath: worktreeSession.worktreePath,
      worktreeBranch: worktreeSession.worktreeBranch,
      message: `Created worktree at ${worktreeSession.worktreePath}${branchInfo}`,
    },
  }
}
```

**执行步骤详解：**

| 步骤 | 行号 | 操作 | 设计意图 |
|------|------|------|----------|
| 1 | 78-81 | 检查当前状态 | 防止嵌套 Worktree |
| 2 | 83-88 | 定位主仓库 | 确保从正确的 Git 根目录创建 |
| 3 | 90 | 获取名称 | 支持用户指定或自动生成 |
| 4 | 92 | 创建 Worktree | 核心创建逻辑 |
| 5 | 94-96 | 切换目录 | 会话上下文切换 |
| 6 | 97 | 状态持久化 | 支持 --resume 恢复 |
| 7 | 99-102 | 清除缓存 | 强制重新计算系统提示 |
| 8 | 104-106 | 分析日志 | 使用追踪 |

### 3.2 createWorktreeForSession 核心实现

```typescript
// 文件：utils/worktree.ts (第 702-778 行)

export async function createWorktreeForSession(
  sessionId: string,
  slug: string,
  tmuxSessionName?: string,
  options?: { prNumber?: number },
): Promise<WorktreeSession> {
  // [第 710 行] 验证名称安全性
  validateWorktreeSlug(slug)

  const originalCwd = getCwd()

  // [第 715-728 行] Hook 模式优先
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
    // [第 729-769 行] Git Worktree 模式
    const gitRoot = findGitRoot(getCwd())
    if (!gitRoot) {
      throw new Error('Cannot create a worktree: not in a git repository')
    }

    const originalBranch = await getBranch()
    const createStart = Date.now()
    
    // 创建或复用 Worktree
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
      originalBranch,
      originalHeadCommit: headCommit,
      sessionId,
      creationDurationMs: existed ? undefined : Date.now() - createStart,
    }
  }

  // [第 772-776 行] 持久化到项目配置
  saveCurrentProjectConfig(current => ({
    ...current,
    activeWorktreeSession: currentWorktreeSession ?? undefined,
  }))

  return currentWorktreeSession
}
```

### 3.3 双模式创建机制

| 模式 | 触发条件 | 执行路径 | 适用场景 |
|------|----------|----------|----------|
| **Hook 模式** | `hasWorktreeCreateHook()` 为 true | `executeWorktreeCreateHook()` | 非 Git VCS、自定义隔离方案 |
| **Git 模式** | 无 Hook 配置 + Git 仓库 | `getOrCreateWorktree()` | 标准 Git 项目 |

---

## 4. 分支管理

### 4.1 分支命名规则

```typescript
// 文件：utils/worktree.ts (第 208-227 行)

// 嵌套名称扁平化：`user/feature` → `user+feature`
function flattenSlug(slug: string): string {
  return slug.replaceAll('/', '+')
}

// 分支名称格式：`worktree-{扁平化名称}`
export function worktreeBranchName(slug: string): string {
  return `worktree-${flattenSlug(slug)}`
}

// Worktree 路径：`.claude/worktrees/{扁平化名称}`
function worktreePathFor(repoRoot: string, slug: string): string {
  return join(worktreesDir(repoRoot), flattenSlug(slug))
}
```

**命名转换示例：**

| 输入名称 | 扁平化后 | 分支名 | 路径 |
|----------|----------|--------|------|
| `feature` | `feature` | `worktree-feature` | `.claude/worktrees/feature` |
| `user/feature` | `user+feature` | `worktree-user+feature` | `.claude/worktrees/user+feature` |
| `bugfix/critical` | `bugfix+critical` | `worktree-bugfix+critical` | `.claude/worktrees/bugfix+critical` |

**扁平化设计原因：**

```typescript
// 文件：utils/worktree.ts (第 208-218 行注释)
// 嵌套在两个位置都不安全：
//   - git refs: `worktree-user` (文件) vs `worktree-user/feature` (需要目录)
//     是 D/F (目录/文件) 冲突，git 会拒绝
//   - 目录: `.claude/worktrees/user/feature/` 在 `user` worktree 内部；
//     `git worktree remove` 删除父级时会连带删除子级未提交的工作
// `+` 在 git 分支名和文件系统路径中都有效，但在 slug 白名单中无效
// ([a-zA-Z0-9._-])，因此映射是单向的（injective）
```

### 4.2 Git Worktree 创建流程

```typescript
// 文件：utils/worktree.ts (第 235-375 行)

async function getOrCreateWorktree(
  repoRoot: string,
  slug: string,
  options?: { prNumber?: number },
): Promise<WorktreeCreateResult> {
  const worktreePath = worktreePathFor(repoRoot, slug)
  const worktreeBranch = worktreeBranchName(slug)

  // [第 247-255 行] 快速恢复路径：检查现有 Worktree
  const existingHead = await readWorktreeHeadSha(worktreePath)
  if (existingHead) {
    return { worktreePath, worktreeBranch, headCommit: existingHead, existed: true }
  }

  // [第 258-303 行] 获取基础分支
  await mkdir(worktreesDir(repoRoot), { recursive: true })
  
  // 环境变量防止 Git 凭据提示（会挂起 CLI）
  const fetchEnv = { ...process.env, ...GIT_NO_PROMPT_ENV }
  
  let baseBranch: string
  if (options?.prNumber) {
    // PR 模式：fetch PR head
    await execFileNoThrowWithCwd(gitExe(), ['fetch', 'origin', `pull/${options.prNumber}/head`], ...)
    baseBranch = 'FETCH_HEAD'
  } else {
    // 智能 fetch：仅当 origin/branch 不存在本地时才 fetch
    const defaultBranch = await getDefaultBranch()
    const originSha = await resolveRef(gitDir, `refs/remotes/origin/${defaultBranch}`)
    baseBranch = originSha ? `origin/${defaultBranch}` : (fetch 成功 ? originRef : 'HEAD')
  }

  // [第 321-334 行] 创建 Worktree
  const addArgs = ['worktree', 'add', '-B', worktreeBranch, worktreePath, baseBranch]
  await execFileNoThrowWithCwd(gitExe(), addArgs, { cwd: repoRoot })

  // [第 336-366 行] Sparse Checkout 处理（可选）
  if (sparsePaths?.length) {
    await execFileNoThrowWithCwd(gitExe(), ['sparse-checkout', 'set', '--cone', '--', ...sparsePaths], ...)
    await execFileNoThrowWithCwd(gitExe(), ['checkout', 'HEAD'], ...)
  }

  return { worktreePath, worktreeBranch, headCommit: baseSha, baseBranch, existed: false }
}
```

### 4.3 关键 Git 参数解析

| 参数 | 作用 | 设计考量 |
|------|------|----------|
| `-B` 而非 `-b` | 强制重置已存在的孤立分支 | 避免 `git branch -D` 子进程开销 |
| `GIT_TERMINAL_PROMPT=0` | 禁止凭据提示 | 防止 CLI 挂起 |
| `GIT_ASKPASS=''` | 禁用 GUI 凭据程序 | 同上 |
| `stdin: 'ignore'` | 关闭 stdin | 阻止交互式提示阻塞 |

---

## 5. 隔离机制

### 5.1 目录隔离

```
主仓库/
├── .claude/
│   ├── worktrees/
│   │   ├── feature-1/       # Worktree 1
│   │   │   ├── .git         # 指向主仓库 .git 的链接文件
│   │   │   └── ...          # 工作文件
│   │   └── feature-2/       # Worktree 2
│   │       ├── .git
│   │       └── ...
│   └── settings.local.json
├── .git/
│   ├── worktrees/           # Git 管理的 worktree 元数据
│   └── config               # 共享配置
└── ...
```

### 5.2 后创建设置（Post Creation Setup）

```typescript
// 文件：utils/worktree.ts (第 510-589 行)

async function performPostCreationSetup(
  repoRoot: string,
  worktreePath: string,
): Promise<void> {
  // [第 514-534 行] 复制 settings.local.json
  const sourceSettingsLocal = join(repoRoot, localSettingsRelativePath)
  await copyFile(sourceSettingsLocal, destSettingsLocal)
  
  // [第 536-578 行] 配置 Git Hooks 路径
  // 解决 .husky 等使用相对路径的 hooks 问题
  const huskyPath = join(repoRoot, '.husky')
  const gitHooksPath = join(repoRoot, '.git', 'hooks')
  // 设置 core.hooksPath 指向主仓库的 hooks
  
  // [第 580-585 行] 符号链接目录（可选）
  const dirsToSymlink = settings.worktree?.symlinkDirectories ?? []
  await symlinkDirectories(repoRoot, worktreePath, dirsToSymlink)
  
  // [第 587-588 行] 复制 .worktreeinclude 文件
  await copyWorktreeIncludeFiles(repoRoot, worktreePath)
}
```

**隔离设置详解：**

| 设置项 | 文件位置 | 目的 |
|--------|----------|------|
| `settings.local.json` 复制 | `.claude/settings.local.json` | 传播本地设置（可能含密钥） |
| `core.hooksPath` 配置 | `.git/config` | 让 Worktree 使用主仓库的 hooks |
| 目录符号链接 | 用户配置 | 防止 `node_modules` 等大目录膨胀 |
| `.worktreeinclude` 复制 | 仓库根目录 | 复制 gitignore 文件（如 `.env`） |

### 5.3 Sparse Checkout 支持

```typescript
// settings.json 配置示例
{
  "worktree": {
    "sparsePaths": ["src/", "package.json", "tsconfig.json"]
  }
}
```

**Sparse Checkout 流程：**

1. 使用 `--no-checkout` 创建空 Worktree
2. 配置 `git sparse-checkout set --cone`
3. 执行 `git checkout HEAD` 填充指定路径

**性能优势：**

| 场景 | 传统 Checkout | Sparse Checkout |
|------|----------------|-----------------|
| 210k 文件仓库 | 全量检出 (~30s) | 仅检出指定目录 (~2s) |
| 磁盘占用 | 全仓库大小 | 仅指定目录大小 |

### 5.4 会话状态隔离

```typescript
// 文件：utils/worktree.ts (第 140-154 行)

export type WorktreeSession = {
  originalCwd: string           // 原始工作目录
  worktreePath: string          // Worktree 路径
  worktreeName: string          // Worktree 名称
  worktreeBranch?: string       // Worktree 分支
  originalBranch?: string       // 原始分支
  originalHeadCommit?: string   // 原始 HEAD commit
  sessionId: string             // 会话 ID
  tmuxSessionName?: string      // tmux 会话名
  hookBased?: boolean           // 是否使用 Hook 模式
  creationDurationMs?: number   // 创建耗时
  usedSparsePaths?: boolean     // 是否使用 Sparse Checkout
}
```

---

## 6. 用户使用指南

### 6.1 使用时机判断

```typescript
// 文件：tools/EnterWorktreeTool/prompt.ts

// 使用时机：
// - 用户明确说 "worktree"（如 "start a worktree", "work in a worktree"）

// 不使用时机：
// - 用户要求创建分支、切换分支 —— 使用 git 命令
// - 用户要求修复 bug 或开发功能 —— 使用正常 git 流程
// - 用户未明确提及 "worktree"
```

### 6.2 UI 呈现

```typescript
// 文件：tools/EnterWorktreeTool/UI.tsx

export function renderToolUseMessage(): React.ReactNode {
  return 'Creating worktree…';
}

export function renderToolResultMessage(output: Output, ...): React.ReactNode {
  return (
    <Box flexDirection="column">
      <Text>
        Switched to worktree on branch <Text bold>{output.worktreeBranch}</Text>
      </Text>
      <Text dimColor>{output.worktreePath}</Text>
    </Box>
  );
}
```

### 6.3 用户交互示例

```
用户: "我想在一个 worktree 中测试这个功能"
Claude: [使用 EnterWorktreeTool]

输出:
  Creating worktree…
  
  Switched to worktree on branch worktree-test-feature
  /path/to/repo/.claude/worktrees/test-feature
```

### 6.4 退出 Worktree

| 方式 | 触发点 | 结果 |
|------|--------|------|
| `ExitWorktreeTool` | 会话中途 | 保留或删除 Worktree（用户选择） |
| 会话结束 | 自然退出 | 提示用户选择保留或删除 |

---

## 7. 从零实现

### 7.1 完整实现代码

```typescript
// === 文件：tools/EnterWorktreeTool/constants.ts ===
export const ENTER_WORKTREE_TOOL_NAME = 'EnterWorktree'

// === 文件：tools/EnterWorktreeTool/prompt.ts ===
export function getEnterWorktreeToolPrompt(): string {
  return `Use this tool ONLY when the user explicitly asks to work in a worktree.
  
## When to Use
- The user explicitly says "worktree"

## When NOT to Use
- The user asks to create/switch branches — use git commands
- Never use unless user explicitly mentions "worktree"

## Requirements
- Must be in a git repository OR have WorktreeCreate hooks configured
- Must not already be in a worktree

## Behavior
- Creates git worktree in .claude/worktrees/ with new branch based on HEAD
- Switches session's working directory to the worktree
- Use ExitWorktree to leave mid-session

## Parameters
- name (optional): Worktree name. Random name generated if not provided.
`
}

// === 文件：tools/EnterWorktreeTool/EnterWorktreeTool.ts ===
import { z } from 'zod/v4'
import { getSessionId, setOriginalCwd } from '../../bootstrap/state.js'
import { clearSystemPromptSections } from '../../constants/systemPromptSections.js'
import { logEvent } from '../../services/analytics/index.js'
import type { Tool } from '../../Tool.js'
import { buildTool, type ToolDef } from '../../Tool.js'
import { clearMemoryFileCaches } from '../../utils/claudemd.js'
import { getCwd } from '../../utils/cwd.js'
import { findCanonicalGitRoot } from '../../utils/git.js'
import { lazySchema } from '../../utils/lazySchema.js'
import { getPlanSlug, getPlansDirectory } from '../../utils/plans.js'
import { setCwd } from '../../utils/Shell.js'
import { saveWorktreeState } from '../../utils/sessionStorage.js'
import {
  createWorktreeForSession,
  getCurrentWorktreeSession,
  validateWorktreeSlug,
} from '../../utils/worktree.js'
import { ENTER_WORKTREE_TOOL_NAME } from './constants.js'
import { getEnterWorktreeToolPrompt } from './prompt.js'
import { renderToolResultMessage, renderToolUseMessage } from './UI.js'

// 延迟初始化 Schema
const inputSchema = lazySchema(() =>
  z.strictObject({
    name: z
      .string()
      .superRefine((s, ctx) => {
        try {
          validateWorktreeSlug(s)
        } catch (e) {
          ctx.addIssue({ code: 'custom', message: (e as Error).message })
        }
      })
      .optional()
      .describe('Optional name for the worktree...'),
  }),
)

const outputSchema = lazySchema(() =>
  z.object({
    worktreePath: z.string(),
    worktreeBranch: z.string().optional(),
    message: z.string(),
  }),
)

type InputSchema = ReturnType<typeof inputSchema>
type OutputSchema = ReturnType<typeof outputSchema>
export type Output = z.infer<OutputSchema>

export const EnterWorktreeTool: Tool<InputSchema, Output> = buildTool({
  name: ENTER_WORKTREE_TOOL_NAME,
  searchHint: 'create an isolated git worktree and switch into it',
  maxResultSizeChars: 100_000,
  
  async description() {
    return 'Creates an isolated worktree and switches the session into it'
  },
  
  async prompt() {
    return getEnterWorktreeToolPrompt()
  },
  
  get inputSchema(): InputSchema {
    return inputSchema()
  },
  
  get outputSchema(): OutputSchema {
    return outputSchema()
  },
  
  userFacingName() {
    return 'Creating worktree'
  },
  
  shouldDefer: true,  // 延迟执行权限检查
  
  toAutoClassifierInput(input) {
    return input.name ?? ''
  },
  
  renderToolUseMessage,
  renderToolResultMessage,
  
  async call(input) {
    // 1. 检查是否已在 worktree 中
    if (getCurrentWorktreeSession()) {
      throw new Error('Already in a worktree session')
    }

    // 2. 定位主仓库根目录
    const mainRepoRoot = findCanonicalGitRoot(getCwd())
    if (mainRepoRoot && mainRepoRoot !== getCwd()) {
      process.chdir(mainRepoRoot)
      setCwd(mainRepoRoot)
    }

    // 3. 获取 worktree 名称
    const slug = input.name ?? getPlanSlug()

    // 4. 创建 worktree
    const worktreeSession = await createWorktreeForSession(getSessionId(), slug)

    // 5. 切换工作目录
    process.chdir(worktreeSession.worktreePath)
    setCwd(worktreeSession.worktreePath)
    setOriginalCwd(getCwd())
    
    // 6. 持久化状态
    saveWorktreeState(worktreeSession)
    
    // 7. 清除缓存
    clearSystemPromptSections()
    clearMemoryFileCaches()
    getPlansDirectory.cache.clear?.()

    // 8. 记录分析
    logEvent('tengu_worktree_created', { mid_session: true })

    // 9. 返回结果
    const branchInfo = worktreeSession.worktreeBranch
      ? ` on branch ${worktreeSession.worktreeBranch}`
      : ''

    return {
      data: {
        worktreePath: worktreeSession.worktreePath,
        worktreeBranch: worktreeSession.worktreeBranch,
        message: `Created worktree at ${worktreeSession.worktreePath}${branchInfo}. Use ExitWorktree to leave.`,
      },
    }
  },
  
  mapToolResultToToolResultBlockParam({ message }, toolUseID) {
    return {
      type: 'tool_result',
      content: message,
      tool_use_id: toolUseID,
    }
  },
} satisfies ToolDef<InputSchema, Output>)

// === 文件：tools/EnterWorktreeTool/UI.tsx ===
import * as React from 'react';
import { Box, Text } from '../../ink.js';
import type { Output } from './EnterWorktreeTool.js';

export function renderToolUseMessage(): React.ReactNode {
  return 'Creating worktree…';
}

export function renderToolResultMessage(
  output: Output,
  _progressMessagesForMessage: ProgressMessage<ToolProgressData>[],
  _options: { theme: ThemeName },
): React.ReactNode {
  return (
    <Box flexDirection="column">
      <Text>
        Switched to worktree on branch <Text bold>{output.worktreeBranch}</Text>
      </Text>
      <Text dimColor>{output.worktreePath}</Text>
    </Box>
  );
}
```

### 7.2 核心 Worktree 创建函数实现

```typescript
// === utils/worktree.ts 核心片段 ===

const VALID_WORKTREE_SLUG_SEGMENT = /^[a-zA-Z0-9._-]+$/
const MAX_WORKTREE_SLUG_LENGTH = 64

const GIT_NO_PROMPT_ENV = {
  GIT_TERMINAL_PROMPT: '0',
  GIT_ASKPASS: '',
}

function worktreesDir(repoRoot: string): string {
  return join(repoRoot, '.claude', 'worktrees')
}

function flattenSlug(slug: string): string {
  return slug.replaceAll('/', '+')
}

export function worktreeBranchName(slug: string): string {
  return `worktree-${flattenSlug(slug)}`
}

export function validateWorktreeSlug(slug: string): void {
  if (slug.length > MAX_WORKTREE_SLUG_LENGTH) {
    throw new Error(`Invalid worktree name: must be 64 characters or fewer`)
  }
  for (const segment of slug.split('/')) {
    if (segment === '.' || segment === '..') {
      throw new Error(`Invalid worktree name: must not contain "." or ".."`)
    }
    if (!VALID_WORKTREE_SLUG_SEGMENT.test(segment)) {
      throw new Error(`Invalid worktree name: invalid characters`)
    }
  }
}

async function getOrCreateWorktree(
  repoRoot: string,
  slug: string,
  options?: { prNumber?: number },
): Promise<WorktreeCreateResult> {
  const worktreePath = join(worktreesDir(repoRoot), flattenSlug(slug))
  const worktreeBranch = `worktree-${flattenSlug(slug)}`

  // 快速恢复：检查是否已存在
  const existingHead = await readWorktreeHeadSha(worktreePath)
  if (existingHead) {
    return { worktreePath, worktreeBranch, headCommit: existingHead, existed: true }
  }

  // 创建新 worktree
  await mkdir(worktreesDir(repoRoot), { recursive: true })
  
  // 获取基础分支
  const defaultBranch = await getDefaultBranch()
  const baseBranch = `origin/${defaultBranch}`
  
  // 执行 git worktree add
  const addArgs = ['worktree', 'add', '-B', worktreeBranch, worktreePath, baseBranch]
  await execFileNoThrowWithCwd(gitExe(), addArgs, { 
    cwd: repoRoot,
    stdin: 'ignore',
    env: { ...process.env, ...GIT_NO_PROMPT_ENV }
  })

  return { worktreePath, worktreeBranch, headCommit: baseSha, baseBranch, existed: false }
}

export async function createWorktreeForSession(
  sessionId: string,
  slug: string,
): Promise<WorktreeSession> {
  validateWorktreeSlug(slug)
  
  const originalCwd = getCwd()
  const gitRoot = findGitRoot(getCwd())
  
  if (!gitRoot) {
    throw new Error('Cannot create worktree: not in a git repository')
  }

  const originalBranch = await getBranch()
  const { worktreePath, worktreeBranch, headCommit, existed } =
    await getOrCreateWorktree(gitRoot, slug)

  if (!existed) {
    await performPostCreationSetup(gitRoot, worktreePath)
  }

  const session: WorktreeSession = {
    originalCwd,
    worktreePath,
    worktreeName: slug,
    worktreeBranch,
    originalBranch,
    originalHeadCommit: headCommit,
    sessionId,
  }

  saveCurrentProjectConfig(current => ({
    ...current,
    activeWorktreeSession: session,
  }))

  return session
}
```

### 7.3 实现要点总结

| 要点 | 实现细节 | 重要性 |
|------|----------|--------|
| **延迟 Schema** | `lazySchema` 包装 | 避免模块加载时的 Zod 开销 |
| **安全验证** | `validateWorktreeSlug` | 防止路径遍历攻击 |
| **名称扁平化** | `/` → `+` 转换 | 避免 Git D/F 冲突 |
| **无凭据提示** | `GIT_NO_PROMPT_ENV` | 防止 CLI 挂起 |
| **快速恢复** | 检查现有 Worktree | 避免重复 fetch |
| **状态持久化** | `saveWorktreeState` | 支持 --resume |
| **缓存清除** | `clearSystemPromptSections` | 强制上下文刷新 |

---

## 附录：关键函数索引

| 函数名 | 文件位置 | 功能 |
|--------|----------|------|
| `validateWorktreeSlug` | `utils/worktree.ts:66` | 名称安全验证 |
| `createWorktreeForSession` | `utils/worktree.ts:702` | 会话 Worktree 创建 |
| `getOrCreateWorktree` | `utils/worktree.ts:235` | Git Worktree 创建/复用 |
| `performPostCreationSetup` | `utils/worktree.ts:510` | 后创建设置 |
| `flattenSlug` | `utils/worktree.ts:217` | 名称扁平化 |
| `worktreeBranchName` | `utils/worktree.ts:221` | 分支名生成 |
| `hasWorktreeCreateHook` | `utils/hooks.ts:4910` | Hook 配置检查 |
| `executeWorktreeCreateHook` | `utils/hooks.ts:4928` | Hook 执行 |
| `saveWorktreeState` | `utils/sessionStorage.ts:2889` | 状态持久化 |
| `lazySchema` | `utils/lazySchema.ts:5` | 延迟 Schema 构造 |
| `buildTool` | `Tool.ts:783` | 工具构建工厂 |

---

*文档生成时间: 2026-04-02*