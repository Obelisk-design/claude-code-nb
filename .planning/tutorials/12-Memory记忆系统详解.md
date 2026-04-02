# Memory记忆系统详解

**分析日期:** 2026-04-02

---

## 1. 概述

### 1.1 Memory系统设计目标

Claude Code的Memory系统是一个**持久化、文件基础的记忆机制**，旨在让AI助手跨对话保持对用户的认知和理解。核心设计目标包括：

1. **跨会话持久性** - 记忆在不同对话间保留，形成对用户的累积认知
2. **分层存储架构** - 区分短期记忆（会话级）和长期记忆（跨会话级）
3. **类型化约束** - 四种封闭类型（user/feedback/project/reference），避免记忆膨胀
4. **团队共享能力** - 支持组织级记忆同步，实现协作知识共享
5. **安全防护** - 密钥扫描、路径验证等多层安全机制

### 1.2 CLAUDE.md文件的作用

CLAUDE.md是**指令加载系统**的核心载体，与Memory系统形成互补关系：

| 文件类型 | 存储位置 | 作用 | 加载优先级 |
|---------|---------|------|-----------|
| Managed CLAUDE.md | `/etc/claude-code/CLAUDE.md` | 全局策略指令（管理员配置） | 最低 |
| User CLAUDE.md | `~/.claude/CLAUDE.md` | 用户私有全局指令 | 低 |
| Project CLAUDE.md | 项目根目录 | 项目级指令（可提交git） | 中 |
| Local CLAUDE.md | `CLAUDE.local.md` | 用户私有项目指令 | 高 |
| `.claude/rules/*.md` | `.claude/rules/` | 条件规则文件 | 与Project同级 |
| MEMORY.md | `~/.claude/projects/<slug>/memory/` | 自动记忆索引 | 系统注入 |

**关键区别：**
- **CLAUDE.md系列** = 静态指令（用户显式编写的行为规范）
- **MEMORY.md** = 动态记忆（AI自动累积的上下文知识）

### 1.3 短期记忆 vs 长期记忆

**短期记忆（会话级）：**
- 当前对话的上下文窗口
- Tasks系统（任务追踪）
- Plan系统（实施计划）
- 随对话结束而消失

**长期记忆（跨会话级）：**
- 自动记忆（AutoMem）- 用户私有，跨项目
- 团队记忆（TeamMem）- 组织共享，项目级
- 存储路径：
  ```
  ~/.claude/projects/<sanitized-git-root>/memory/         # AutoMem
  ~/.claude/projects/<sanitized-git-root>/memory/team/    # TeamMem
  ```

---

## 2. memdir/ 目录逐行分析

### 2.1 memdir/memdir.ts 核心实现

**文件路径:** `memdir/memdir.ts`

**核心导出：**

```typescript
// 常量定义
export const ENTRYPOINT_NAME = 'MEMORY.md'       // 记忆索引文件名
export const MAX_ENTRYPOINT_LINES = 200          // 最大行数限制
export const MAX_ENTRYPOINT_BYTES = 25_000       // 最大字节限制（~25KB）
```

#### truncateEntrypointContent() - 内容截断函数（行57-103）

```typescript
export function truncateEntrypointContent(raw: string): EntrypointTruncation {
  const trimmed = raw.trim()
  const contentLines = trimmed.split('\n')
  const lineCount = contentLines.length
  const byteCount = trimmed.length

  // 检测是否需要截断
  const wasLineTruncated = lineCount > MAX_ENTRYPOINT_LINES
  const wasByteTruncated = byteCount > MAX_ENTRYPOINT_BYTES

  if (!wasLineTruncated && !wasByteTruncated) {
    return { content: trimmed, lineCount, byteCount, wasLineTruncated, wasByteTruncated }
  }

  // 先按行截断
  let truncated = wasLineTruncated
    ? contentLines.slice(0, MAX_ENTRYPOINT_LINES).join('\n')
    : trimmed

  // 再按字节截断（在最后一个换行符处截断，避免切断行）
  if (truncated.length > MAX_ENTRYPOINT_BYTES) {
    const cutAt = truncated.lastIndexOf('\n', MAX_ENTRYPOINT_BYTES)
    truncated = truncated.slice(0, cutAt > 0 ? cutAt : MAX_ENTRYPOINT_BYTES)
  }

  // 添加警告信息
  const reason = wasByteTruncated && !wasLineTruncated
    ? `${formatFileSize(byteCount)} (limit: ${formatFileSize(MAX_ENTRYPOINT_BYTES)}) — index entries are too long`
    : wasLineTruncated && !wasByteTruncated
      ? `${lineCount} lines (limit: ${MAX_ENTRYPOINT_LINES})`
      : `${lineCount} lines and ${formatFileSize(byteCount)}`

  return {
    content: truncated + `\n\n> WARNING: ${ENTRYPOINT_NAME} is ${reason}. Only part of it was loaded.`,
    lineCount,
    byteCount,
    wasLineTruncated,
    wasByteTruncated,
  }
}
```

**设计意图：**
- 双重限制（行数 + 字节）防止记忆文件膨胀
- 行优先截断保持内容结构完整性
- 字节截断在换行处切割，避免半行内容

#### buildMemoryLines() - 构建记忆提示词（行199-266）

```typescript
export function buildMemoryLines(
  displayName: string,
  memoryDir: string,
  extraGuidelines?: string[],
  skipIndex = false,
): string[] {
  const howToSave = skipIndex
    ? [
        '## How to save memories',
        '',
        'Write each memory to its own file using this frontmatter format:',
        '',
        ...MEMORY_FRONTMATTER_EXAMPLE,
        '',
        '- Keep the name, description, and type fields up-to-date',
        '- Organize memory semantically by topic, not chronologically',
        '- Update or remove outdated memories',
        '- Do not write duplicate memories',
      ]
    : [
        '## How to save memories',
        '',
        'Saving a memory is a two-step process:',
        '',
        '**Step 1** — write to its own file using frontmatter format',
        '**Step 2** — add pointer in MEMORY.md index',
        '',
        `- MEMORY.md is always loaded — lines after ${MAX_ENTRYPOINT_LINES} will be truncated`,
      ]

  const lines: string[] = [
    `# ${displayName}`,
    '',
    `You have a persistent, file-based memory system at \`${memoryDir}\`. ${DIR_EXISTS_GUIDANCE}`,
    '',
    "Build up this memory system so future conversations have a complete picture of who the user is.",
    '',
    'If the user explicitly asks to remember something, save it immediately.',
    '',
    ...TYPES_SECTION_INDIVIDUAL,      // 四种记忆类型定义
    ...WHAT_NOT_TO_SAVE_SECTION,       // 不应保存的内容清单
    '',
    ...howToSave,
    '',
    ...WHEN_TO_ACCESS_SECTION,         // 访问时机指南
    '',
    ...TRUSTING_RECALL_SECTION,        // 信任度评估指南
    '',
    '## Memory and other forms of persistence',
    // ...区分memory vs plan vs tasks的使用场景
  ]

  return lines
}
```

**关键概念：**
- **两步保存流程**：先写单独文件，再更新MEMORY.md索引
- **语义组织**：按主题而非时间组织记忆
- **去重原则**：检查现有记忆再创建新记忆

#### loadMemoryPrompt() - 加载记忆提示词（行419-507）

```typescript
export async function loadMemoryPrompt(): Promise<string | null> {
  const autoEnabled = isAutoMemoryEnabled()
  const skipIndex = getFeatureValue_CACHED_MAY_BE_STALE('tengu_moth_copse', false)

  // KAIROS模式：助手的每日日志模式
  if (feature('KAIROS') && autoEnabled && getKairosActive()) {
    logMemoryDirCounts(getAutoMemPath(), { memory_type: 'auto' })
    return buildAssistantDailyLogPrompt(skipIndex)
  }

  // 获取Cowork额外指南
  const coworkExtraGuidelines = process.env.CLAUDE_COWORK_MEMORY_EXTRA_GUIDELINES
  const extraGuidelines = coworkExtraGuidelines?.trim().length > 0
    ? [coworkExtraGuidelines]
    : undefined

  // TEAMMEM模式：团队记忆 + 自动记忆组合
  if (feature('TEAMMEM')) {
    if (teamMemPaths!.isTeamMemoryEnabled()) {
      const autoDir = getAutoMemPath()
      const teamDir = teamMemPaths!.getTeamMemPath()
      
      await ensureMemoryDirExists(teamDir)
      
      logMemoryDirCounts(autoDir, { memory_type: 'auto' })
      logMemoryDirCounts(teamDir, { memory_type: 'team' })
      
      return teamMemPrompts!.buildCombinedMemoryPrompt(extraGuidelines, skipIndex)
    }
  }

  // 仅AutoMem模式
  if (autoEnabled) {
    const autoDir = getAutoMemPath()
    await ensureMemoryDirExists(autoDir)
    
    return buildMemoryLines('auto memory', autoDir, extraGuidelines, skipIndex).join('\n')
  }

  // 记录禁用原因
  logEvent('tengu_memdir_disabled', {
    disabled_by_env_var: isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_AUTO_MEMORY),
    disabled_by_setting: getInitialSettings().autoMemoryEnabled === false,
  })
  
  return null
}
```

**加载模式优先级：**
1. KAIROS每日日志模式（助手专用）
2. TEAMMEM组合模式（团队 + 私有）
3. 仅AutoMem模式（私有记忆）
4. 禁用状态（返回null）

---

### 2.2 memdir/paths.ts 路径管理

**文件路径:** `memdir/paths.ts`

#### isAutoMemoryEnabled() - 功能开关检查（行30-55）

```typescript
export function isAutoMemoryEnabled(): boolean {
  // 优先级链（第一个定义的获胜）
  
  // 1. 环境变量覆盖
  const envVal = process.env.CLAUDE_CODE_DISABLE_AUTO_MEMORY
  if (isEnvTruthy(envVal)) return false
  if (isEnvDefinedFalsy(envVal)) return true
  
  // 2. SIMPLE模式（--bare）
  if (isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)) return false
  
  // 3. CCR无持久存储
  if (isEnvTruthy(process.env.CLAUDE_CODE_REMOTE) &&
      !process.env.CLAUDE_CODE_REMOTE_MEMORY_DIR) {
    return false
  }
  
  // 4. settings.json配置
  const settings = getInitialSettings()
  if (settings.autoMemoryEnabled !== undefined) {
    return settings.autoMemoryEnabled
  }
  
  // 5. 默认启用
  return true
}
```

**禁用条件优先级：**
```
环境变量 > SIMPLE模式 > CCR模式 > settings.json > 默认启用
```

#### validateMemoryPath() - 路径安全验证（行109-150）

```typescript
function validateMemoryPath(
  raw: string | undefined,
  expandTilde: boolean,
): string | undefined {
  if (!raw) return undefined
  
  let candidate = raw
  
  // ~/ 路径展开（仅settings.json支持）
  if (expandTilde && (candidate.startsWith('~/') || candidate.startsWith('~\\'))) {
    const rest = candidate.slice(2)
    const restNorm = normalize(rest || '.')
    
    // 拒绝危险展开（展开到$HOME或父目录）
    if (restNorm === '.' || restNorm === '..') return undefined
    
    candidate = join(homedir(), rest)
  }
  
  // 路径规范化
  const normalized = normalize(candidate).replace(/[/\\]+$/, '')
  
  // 安全检查
  if (
    !isAbsolute(normalized) ||           // 相对路径
    normalized.length < 3 ||             // 根路径/短路径
    /^[A-Za-z]:$/.test(normalized) ||    // Windows驱动器根
    normalized.startsWith('\\\\') ||      // UNC路径
    normalized.startsWith('//') ||        // UNC路径
    normalized.includes('\0')             // null字节注入
  ) {
    return undefined
  }
  
  return (normalized + sep).normalize('NFC')
}
```

**安全拒绝条件：**
- 相对路径（`../foo`）
- 根路径（`/`、`C:`）
- UNC网络路径（`\\server\share`）
- null字节注入攻击
- Unicode规范化攻击

#### getAutoMemPath() - 获取记忆目录路径（行223-235）

```typescript
export const getAutoMemPath = memoize(
  (): string => {
    // 优先级：
    // 1. CLAUDE_COWORK_MEMORY_PATH_OVERRIDE环境变量
    // 2. autoMemoryDirectory设置（仅信任来源）
    // 3. 默认：<memoryBase>/projects/<sanitized-git-root>/memory/
    
    const override = getAutoMemPathOverride() ?? getAutoMemPathSetting()
    if (override) return override
    
    const projectsDir = join(getMemoryBaseDir(), 'projects')
    return (
      join(projectsDir, sanitizePath(getAutoMemBase()), AUTO_MEM_DIRNAME) + sep
    ).normalize('NFC')
  },
  () => getProjectRoot(),  // 缓存键
)
```

**路径构建逻辑：**
```
~/.claude/projects/<git-root-hash>/memory/
```

使用`sanitizePath()`确保git根路径安全转换为目录名。

---

### 2.3 memdir/memoryTypes.ts 类型定义

**文件路径:** `memdir/memoryTypes.ts`

#### 四种记忆类型（行14-106）

```typescript
export const MEMORY_TYPES = ['user', 'feedback', 'project', 'reference'] as const
export type MemoryType = (typeof MEMORY_TYPES)[number]
```

**类型详解：**

| 类型 | 作用域 | 用途 | 示例 |
|------|--------|------|------|
| `user` | 仅私有 | 用户角色、偏好、知识背景 | "用户是数据科学家，关注日志分析" |
| `feedback` | 私有或团队 | 行为指导（纠正/确认） | "不要mock数据库，曾有migration事故" |
| `project` | 倾向团队 | 项目动态、决策、约束 | "3月5日起冻结合并，移动发布" |
| `reference` | 通常团队 | 外部系统引用 | "pipeline bugs在Linear INGEST项目" |

#### TYPES_SECTION_COMBINED - 组合模式提示词（行37-106）

```typescript
export const TYPES_SECTION_COMBINED: readonly string[] = [
  '## Types of memory',
  '',
  'There are several discrete types of memory. Each type declares a <scope>:',
  '',
  '<types>',
  '<type>',
  '    <name>user</name>',
  '    <scope>always private</scope>',
  '    <description>Information about user\'s role, goals, responsibilities...</description>',
  '    <when_to_save>When you learn details about user\'s preferences...</when_to_save>',
  '    <how_to_use>Tailor future behavior to user\'s perspective...</how_to_use>',
  '    <examples>',
  '    user: I\'m a data scientist...',
  '    assistant: [saves private user memory: user is a data scientist]',
  '    </examples>',
  '</type>',
  // ...feedback, project, reference类型...
  '</types>',
]
```

**XML风格结构化提示词设计：**
- `<name>` - 类型名称
- `<scope>` - 存储范围（private/team）
- `<description>` - 类型用途
- `<when_to_save>` - 保存时机
- `<how_to_use>` - 使用时机
- `<examples>` - 具体示例

#### WHAT_NOT_TO_SAVE_SECTION - 禁止保存内容（行183-195）

```typescript
export const WHAT_NOT_TO_SAVE_SECTION: readonly string[] = [
  '## What NOT to save in memory',
  '',
  '- Code patterns, conventions, architecture — derivable from current project state',
  '- Git history, recent changes — `git log` / `git blame` are authoritative',
  '- Debugging solutions — the fix is in the code, commit message has context',
  '- Anything already documented in CLAUDE.md files',
  '- Ephemeral task details: in-progress work, temporary state',
  '',
  // 显式保存请求的处理指南
  'These exclusions apply even when the user explicitly asks you to save. ' +
  'If they ask to save a PR list, ask what was *surprising* — that is worth keeping.',
]
```

**核心原则：**
- **可推导内容不保存** - 代码结构、git历史可通过工具获取
- **已有文档不保存** - CLAUDE.md已覆盖的内容
- **临时状态不保存** - 会话级任务用Tasks系统
- **显式请求需过滤** - 用户要求保存时，提取真正有价值的部分

---

### 2.4 memdir/memoryAge.ts 年龄追踪

**文件路径:** `memdir/memoryAge.ts`

#### memoryAgeDays() - 计算记忆年龄（行6-8）

```typescript
export function memoryAgeDays(mtimeMs: number): number {
  // 向下取整天数，未来时间戳钳位到0
  return Math.max(0, Math.floor((Date.now() - mtimeMs) / 86_400_000))
}
```

**86_400_000** = 24小时 × 60分钟 × 60秒 × 1000毫秒

#### memoryAge() - 人类可读年龄字符串（行15-20）

```typescript
export function memoryAge(mtimeMs: number): string {
  const d = memoryAgeDays(mtimeMs)
  if (d === 0) return 'today'
  if (d === 1) return 'yesterday'
  return `${d} days ago`
}
```

**设计意图：**
- 模型不擅长日期算术，"47 days ago"比ISO时间戳更能触发陈旧性推理

#### memoryFreshnessText() - 陈旧性警告文本（行33-42）

```typescript
export function memoryFreshnessText(mtimeMs: number): string {
  const d = memoryAgeDays(mtimeMs)
  if (d <= 1) return ''  // 今天/昨天的记忆无需警告
  
  return (
    `This memory is ${d} days old. ` +
    `Memories are point-in-time observations, not live state — ` +
    `claims about code behavior or file:line citations may be outdated. ` +
    `Verify against current code before asserting as fact.`
  )
}
```

**触发场景：**
- 记忆中引用的代码位置可能已变更
- 陈旧记忆被错误当作当前事实

---

### 2.5 memdir/findRelevantMemories.ts 相关记忆查找

**文件路径:** `memdir/findRelevantMemories.ts`

#### findRelevantMemories() - 查询相关记忆（行39-75）

```typescript
export async function findRelevantMemories(
  query: string,
  memoryDir: string,
  signal: AbortSignal,
  recentTools: readonly string[] = [],
  alreadySurfaced: ReadonlySet<string> = new Set(),
): Promise<RelevantMemory[]> {
  // 扫描记忆文件，排除已显示的
  const memories = (await scanMemoryFiles(memoryDir, signal)).filter(
    m => !alreadySurfaced.has(m.filePath),
  )
  
  if (memories.length === 0) return []

  // 使用Sonnet模型选择最相关的记忆（最多5个）
  const selectedFilenames = await selectRelevantMemories(query, memories, signal, recentTools)
  
  const byFilename = new Map(memories.map(m => [m.filename, m]))
  const selected = selectedFilenames
    .map(filename => byFilename.get(filename))
    .filter((m): m is MemoryHeader => m !== undefined)

  // 遥测记录
  if (feature('MEMORY_SHAPE_TELEMETRY')) {
    logMemoryRecallShape(memories, selected)
  }

  return selected.map(m => ({ path: m.filePath, mtimeMs: m.mtimeMs }))
}
```

**工作流程：**
1. 扫描记忆目录获取所有文件头信息
2. 过滤已在本对话中显示的记忆
3. 调用Sonnet模型进行相关性选择
4. 返回最多5个最相关记忆的路径+修改时间

#### selectRelevantMemories() - 模型选择逻辑（行77-141）

```typescript
async function selectRelevantMemories(
  query: string,
  memories: MemoryHeader[],
  signal: AbortSignal,
  recentTools: readonly string[],
): Promise<string[]> {
  const validFilenames = new Set(memories.map(m => m.filename))
  const manifest = formatMemoryManifest(memories)

  // 添加最近使用工具信息（避免推荐正在使用的工具文档）
  const toolsSection = recentTools.length > 0
    ? `\n\nRecently used tools: ${recentTools.join(', ')}`
    : ''

  const result = await sideQuery({
    model: getDefaultSonnetModel(),
    system: SELECT_MEMORIES_SYSTEM_PROMPT,
    skipSystemPromptPrefix: true,
    messages: [{
      role: 'user',
      content: `Query: ${query}\n\nAvailable memories:\n${manifest}${toolsSection}`,
    }],
    max_tokens: 256,
    output_format: {
      type: 'json_schema',
      schema: {
        type: 'object',
        properties: {
          selected_memories: { type: 'array', items: { type: 'string' } },
        },
        required: ['selected_memories'],
        additionalProperties: false,
      },
    },
    signal,
    querySource: 'memdir_relevance',
  })

  const parsed = jsonParse(textBlock.text)
  return parsed.selected_memories.filter(f => validFilenames.has(f))
}
```

**系统提示词关键约束：**
```typescript
const SELECT_MEMORIES_SYSTEM_PROMPT = `
Return filenames for memories that will clearly be useful (up to 5).
- If unsure, do NOT include — be selective and discerning.
- If no memories would clearly be useful, return empty list.
- Do NOT select usage reference for recently-used tools.
- DO still select warnings/gotchas about those tools.
`
```

---

### 2.6 memdir/memoryScan.ts 记忆扫描

**文件路径:** `memdir/memoryScan.ts`

#### scanMemoryFiles() - 扫描记忆文件（行35-77）

```typescript
export async function scanMemoryFiles(
  memoryDir: string,
  signal: AbortSignal,
): Promise<MemoryHeader[]> {
  try {
    const entries = await readdir(memoryDir, { recursive: true })
    const mdFiles = entries.filter(
      f => f.endsWith('.md') && basename(f) !== 'MEMORY.md',  // 排除索引文件
    )

    // 并行读取所有文件的前30行（frontmatter范围）
    const headerResults = await Promise.allSettled(
      mdFiles.map(async (relativePath): Promise<MemoryHeader> => {
        const filePath = join(memoryDir, relativePath)
        const { content, mtimeMs } = await readFileInRange(
          filePath,
          0,
          FRONTMATTER_MAX_LINES,  // = 30
          undefined,
          signal,
        )
        
        const { frontmatter } = parseFrontmatter(content, filePath)
        
        return {
          filename: relativePath,
          filePath,
          mtimeMs,
          description: frontmatter.description || null,
          type: parseMemoryType(frontmatter.type),
        }
      }),
    )

    // 过滤成功结果，按修改时间降序排序，限制200个文件
    return headerResults
      .filter(r => r.status === 'fulfilled')
      .map(r => r.value)
      .sort((a, b) => b.mtimeMs - a.mtimeMs)
      .slice(0, MAX_MEMORY_FILES)  // = 200
  } catch {
    return []
  }
}
```

**性能优化策略：**
- **单次扫描**：readFileInRange内部stat获取mtime，避免双重syscall
- **并行处理**：Promise.allSettled并行读取所有文件
- **限制数量**：最多200个文件，防止记忆膨胀

#### formatMemoryManifest() - 格式化清单（行84-94）

```typescript
export function formatMemoryManifest(memories: MemoryHeader[]): string {
  return memories
    .map(m => {
      const tag = m.type ? `[${m.type}] ` : ''
      const ts = new Date(m.mtimeMs).toISOString()
      return m.description
        ? `- ${tag}${m.filename} (${ts}): ${m.description}`
        : `- ${tag}${m.filename} (${ts})`
    })
    .join('\n')
}
```

**输出格式示例：**
```
- [feedback] testing-guide.md (2026-03-15T10:30:00Z): Integration tests must hit real database
- [project] merge-freeze.md (2026-03-10T08:00:00Z): Freeze starts 2026-03-05
```

---

### 2.7 memdir/teamMemPaths.ts 团队记忆路径

**文件路径:** `memdir/teamMemPaths.ts`

#### validateTeamMemWritePath() - 写路径验证（行228-256）

```typescript
export async function validateTeamMemWritePath(filePath: string): Promise<string> {
  if (filePath.includes('\0')) {
    throw new PathTraversalError(`Null byte in path: "${filePath}"`)
  }

  // 第一轮：字符串级 containment 检查
  const resolvedPath = resolve(filePath)
  const teamDir = getTeamMemPath()
  
  if (!resolvedPath.startsWith(teamDir)) {
    throw new PathTraversalError(`Path escapes team memory directory: "${filePath}"`)
  }

  // 第二轮：symlink解析 + 真实路径验证
  const realPath = await realpathDeepestExisting(resolvedPath)
  
  if (!(await isRealPathWithinTeamDir(realPath))) {
    throw new PathTraversalError(`Path escapes via symlink: "${filePath}"`)
  }
  
  return resolvedPath
}
```

**两轮验证机制：**
1. **快速字符串检查** - reject明显的`..`遍历
2. **symlink深度解析** - 防止符号链接逃逸攻击

#### realpathDeepestExisting() - 深度解析（行109-171）

```typescript
async function realpathDeepestExisting(absolutePath: string): Promise<string> {
  const tail: string[] = []
  let current = absolutePath
  
  // 向上遍历直到找到存在的祖先
  for (let parent = dirname(current); current !== parent; parent = dirname(current)) {
    try {
      const realCurrent = await realpath(current)
      return tail.length === 0
        ? realCurrent
        : join(realCurrent, ...tail.reverse())
    } catch (e: unknown) {
      const code = getErrnoCode(e)
      
      if (code === 'ENOENT') {
        // 区分：真正不存在 vs dangling symlink
        try {
          const st = await lstat(current)
          if (st.isSymbolicLink()) {
            throw new PathTraversalError(`Dangling symlink: "${current}"`)
          }
        } catch (lstatErr) {
          if (lstatErr instanceof PathTraversalError) throw lstatErr
        }
      } else if (code === 'ELOOP') {
        throw new PathTraversalError(`Symlink loop: "${current}"`)
      }
      
      tail.push(current.slice(parent.length + sep.length))
      current = parent
    }
  }
  
  return absolutePath
}
```

**处理场景：**
- 文件尚不存在（准备写入）
- dangling symlink（目标不存在）
- symlink loop（循环引用）
- 权限错误等其他问题

---

### 2.8 memdir/teamMemPrompts.ts 团队记忆提示词

**文件路径:** `memdir/teamMemPrompts.ts`

#### buildCombinedMemoryPrompt() - 组合提示词（行22-100）

```typescript
export function buildCombinedMemoryPrompt(
  extraGuidelines?: string[],
  skipIndex = false,
): string {
  const autoDir = getAutoMemPath()
  const teamDir = getTeamMemPath()

  const lines = [
    '# Memory',
    '',
    `You have two directories: private at \`${autoDir}\` and team at \`${teamDir}\`.`,
    '',
    '## Memory scope',
    '',
    '- private: memories between you and current user only',
    '- team: memories shared with all project contributors (synced)',
    '',
    ...TYPES_SECTION_COMBINED,      // 带scope标签的类型定义
    ...WHAT_NOT_TO_SAVE_SECTION,
    '- You MUST avoid saving sensitive data in team memories.',
    '',
    ...howToSave,
    '',
    '## When to access memories',
    '- When memories seem relevant or user references prior work',
    '- You MUST access memory when user explicitly asks',
    '- If user says to *ignore* memory: proceed as if MEMORY.md were empty',
    MEMORY_DRIFT_CAVEAT,
    '',
    ...TRUSTING_RECALL_SECTION,
    '',
    '## Memory and other forms of persistence',
    // ...plan vs memory vs tasks区分
    ...(extraGuidelines ?? []),
    '',
    ...buildSearchingPastContextSection(autoDir),
  ]

  return lines.join('\n')
}
```

**关键差异（vs Individual模式）：**
- 双目录提示（私有 + 团队）
- 每个类型带`<scope>`标签
- 示例区分`[saves private/team memory: ...]`
- 团队记忆敏感数据禁令

---

## 3. CLAUDE.md处理机制

### 3.1 utils/claudemd.ts 逐行分析

**文件路径:** `utils/claudemd.ts`

#### 文件加载顺序（文件头注释）

```
1. Managed memory (/etc/claude-code/CLAUDE.md) - 全局策略
2. User memory (~/.claude/CLAUDE.md) - 私有全局
3. Project memory (CLAUDE.md, .claude/CLAUDE.md, .claude/rules/*.md) - 项目级
4. Local memory (CLAUDE.local.md) - 私有项目

加载顺序：越晚加载优先级越高（模型更关注）
```

#### parseMemoryFileContent() - 解析记忆文件（行343-400）

```typescript
function parseMemoryFileContent(
  rawContent: string,
  filePath: string,
  type: MemoryType,
  includeBasePath?: string,
): { info: MemoryFileInfo | null; includePaths: string[] } {
  // 跳过非文本文件
  const ext = extname(filePath).toLowerCase()
  if (ext && !TEXT_FILE_EXTENSIONS.has(ext)) {
    return { info: null, includePaths: [] }
  }

  // 解析frontmatter获取paths字段
  const { content: withoutFrontmatter, paths } = parseFrontmatterPaths(rawContent)

  // 使用marked lexer解析（用于HTML注释移除和@include提取）
  const tokens = hasComment || includeBasePath !== undefined
    ? new Lexer({ gfm: false }).lex(withoutFrontmatter)
    : undefined

  // 移除HTML注释
  const strippedContent = hasComment && tokens
    ? stripHtmlCommentsFromTokens(tokens).content
    : withoutFrontmatter

  // 提取@include路径
  const includePaths = tokens && includeBasePath !== undefined
    ? extractIncludePathsFromTokens(tokens, includeBasePath)
    : []

  // MEMORY.md截断
  let finalContent = strippedContent
  if (type === 'AutoMem' || type === 'TeamMem') {
    finalContent = truncateEntrypointContent(strippedContent).content
  }

  return {
    info: {
      path: filePath,
      type,
      content: finalContent,
      globs: paths,
      contentDiffersFromDisk: finalContent !== rawContent,
      rawContent: contentDiffersFromDisk ? rawContent : undefined,
    },
    includePaths,
  }
}
```

**处理流程：**
1. 文件类型检查（TEXT_FILE_EXTENSIONS）
2. frontmatter解析（提取paths字段）
3. HTML注释移除（保留代码块内注释）
4. @include路径提取
5. MEMORY.md截断（仅AutoMem/TeamMem）

#### stripHtmlComments() - HTML注释移除（行292-334）

```typescript
export function stripHtmlComments(content: string): { content: string; stripped: boolean } {
  if (!content.includes('<!--')) {
    return { content, stripped: false }
  }
  
  return stripHtmlCommentsFromTokens(new Lexer({ gfm: false }).lex(content))
}

function stripHtmlCommentsFromTokens(tokens): { content: string; stripped: boolean } {
  let result = ''
  let stripped = false
  
  const commentSpan = /<!--[\s\S]*?-->/g  // 非贪婪匹配
  
  for (const token of tokens) {
    if (token.type === 'html') {
      const trimmed = token.raw.trimStart()
      if (trimmed.startsWith('<!--') && trimmed.includes('-->')) {
        // 移除注释span，保留残余内容
        const residue = token.raw.replace(commentSpan, '')
        stripped = true
        if (residue.trim().length > 0) {
          result += residue  // 如：`<!-- note --> Use bun`
        }
        continue
      }
    }
    result += token.raw
  }
  
  return { content: result, stripped }
}
```

**设计意图：**
- 仅移除块级注释（独立成行的）
- 保留行内注释和代码块内注释
- 保留注释后的残余文本

#### @include机制（行451-535）

```typescript
function extractIncludePathsFromTokens(tokens, basePath: string): string[] {
  const absolutePaths = new Set<string>()
  
  function extractPathsFromText(textContent: string) {
    const includeRegex = /(?:^|\s)@((?:[^\s\\]|\\ )+)/g
    let match
    
    while ((match = includeRegex.exec(textContent)) !== null) {
      let path = match[1]
      
      // 移除fragment标识符（#heading）
      const hashIndex = path.indexOf('#')
      if (hashIndex !== -1) {
        path = path.substring(0, hashIndex)
      }
      
      // 反转义空格
      path = path.replace(/\\ /g, ' ')
      
      // 验证路径格式
      const isValidPath =
        path.startsWith('./') ||
        path.startsWith('~/') ||
        (path.startsWith('/') && path !== '/') ||
        (!path.startsWith('@') && !path.match(/^[#%^&*()]+/) && path.match(/^[a-zA-Z0-9._-]/))
      
      if (isValidPath) {
        const resolvedPath = expandPath(path, dirname(basePath))
        absolutePaths.add(resolvedPath)
      }
    }
  }
  
  // 递归处理token树
  function processElements(elements) {
    for (const element of elements) {
      if (element.type === 'code' || element.type === 'codespan') continue  // 跳过代码块
      
      if (element.type === 'text') {
        extractPathsFromText(element.text || '')
      }
      
      if (element.tokens) processElements(element.tokens)
      if (element.items) processElements(element.items)
    }
  }
  
  processElements(tokens)
  return [...absolutePaths]
}
```

**支持的路径格式：**
- `@./relative/path.md` - 相对路径
- `@~/home/path.md` - 用户目录
- `@/absolute/path.md` - 绝对路径
- `@path.md` - 等同于`@./path.md`

**排除区域：**
- 代码块（`code`、`codespan`类型）
- HTML注释内容

#### processMemoryFile() - 处理单个文件（行618-685）

```typescript
export async function processMemoryFile(
  filePath: string,
  type: MemoryType,
  processedPaths: Set<string>,
  includeExternal: boolean,
  depth: number = 0,
  parent?: string,
): Promise<MemoryFileInfo[]> {
  // 循环引用检测
  const normalizedPath = normalizePathForComparison(filePath)
  if (processedPaths.has(normalizedPath) || depth >= MAX_INCLUDE_DEPTH) {
    return []
  }
  
  // 排除检查
  if (isClaudeMdExcluded(filePath, type)) {
    return []
  }
  
  // Symlink解析
  const { resolvedPath, isSymlink } = safeResolvePath(getFsImplementation(), filePath)
  processedPaths.add(normalizedPath)
  if (isSymlink) {
    processedPaths.add(normalizePathForComparison(resolvedPath))
  }
  
  // 读取并解析
  const { info: memoryFile, includePaths } =
    await safelyReadMemoryFileAsync(filePath, type, resolvedPath)
  
  if (!memoryFile || !memoryFile.content.trim()) {
    return []
  }
  
  // 设置父文件引用
  if (parent) {
    memoryFile.parent = parent
  }
  
  const result: MemoryFileInfo[] = [memoryFile]
  
  // 递归处理@include
  for (const resolvedIncludePath of includePaths) {
    const isExternal = !pathInOriginalCwd(resolvedIncludePath)
    if (isExternal && !includeExternal) continue
    
    const includedFiles = await processMemoryFile(
      resolvedIncludePath,
      type,
      processedPaths,
      includeExternal,
      depth + 1,
      filePath,  // 当前文件作为父
    )
    result.push(...includedFiles)
  }
  
  return result
}
```

**关键机制：**
- **MAX_INCLUDE_DEPTH = 5** - 防止无限递归
- **processedPaths** - 循环引用检测
- **parent字段** - 记录include链
- **isClaudeMdExcluded** - 用户排除设置

#### getMemoryFiles() - 获取所有记忆文件（行790-1075）

```typescript
export const getMemoryFiles = memoize(
  async (forceIncludeExternal: boolean = false): Promise<MemoryFileInfo[]> => {
    const result: MemoryFileInfo[] = []
    const processedPaths = new Set<string>()
    const includeExternal = forceIncludeExternal || config.hasClaudeMdExternalIncludesApproved
    
    // 1. Managed文件（最高优先级策略）
    result.push(...(await processMemoryFile(
      getMemoryPath('Managed'),
      'Managed',
      processedPaths,
      includeExternal,
    )))
    
    // 2. Managed rules
    result.push(...(await processMdRules({
      rulesDir: getManagedClaudeRulesDir(),
      type: 'Managed',
      processedPaths,
      includeExternal,
      conditionalRule: false,
    })))
    
    // 3. User文件（需要userSettings启用）
    if (isSettingSourceEnabled('userSettings')) {
      result.push(...(await processMemoryFile(
        getMemoryPath('User'),
        'User',
        processedPaths,
        true,  // User memory可包含外部文件
      )))
      
      // User rules
      result.push(...(await processMdRules({
        rulesDir: getUserClaudeRulesDir(),
        type: 'User',
        processedPaths,
        includeExternal: true,
        conditionalRule: false,
      })))
    }
    
    // 4. Project/Local文件（向上遍历）
    const dirs: string[] = []
    let currentDir = getOriginalCwd()
    
    while (currentDir !== parse(currentDir).root) {
      dirs.push(currentDir)
      currentDir = dirname(currentDir)
    }
    
    // 从根向下处理（越近CWD优先级越高）
    for (const dir of dirs.reverse()) {
      // Worktree特殊处理...
      
      if (isSettingSourceEnabled('projectSettings')) {
        result.push(...(await processMemoryFile(
          join(dir, 'CLAUDE.md'),
          'Project',
          processedPaths,
          includeExternal,
        )))
        
        // .claude/CLAUDE.md
        result.push(...(await processMemoryFile(
          join(dir, '.claude', 'CLAUDE.md'),
          'Project',
          processedPaths,
          includeExternal,
        )))
        
        // .claude/rules/*.md
        result.push(...(await processMdRules({
          rulesDir: join(dir, '.claude', 'rules'),
          type: 'Project',
          processedPaths,
          includeExternal,
          conditionalRule: false,
        })))
      }
      
      if (isSettingSourceEnabled('localSettings')) {
        result.push(...(await processMemoryFile(
          join(dir, 'CLAUDE.local.md'),
          'Local',
          processedPaths,
          includeExternal,
        )))
      }
    }
    
    // 5. AutoMem MEMORY.md
    if (isAutoMemoryEnabled()) {
      const { info: memdirEntry } = await safelyReadMemoryFileAsync(
        getAutoMemEntrypoint(),
        'AutoMem',
      )
      if (memdirEntry) {
        const normalizedPath = normalizePathForComparison(memdirEntry.path)
        if (!processedPaths.has(normalizedPath)) {
          processedPaths.add(normalizedPath)
          result.push(memdirEntry)
        }
      }
    }
    
    // 6. TeamMem MEMORY.md
    if (feature('TEAMMEM') && teamMemPaths!.isTeamMemoryEnabled()) {
      const { info: teamMemEntry } = await safelyReadMemoryFileAsync(
        teamMemPaths!.getTeamMemEntrypoint(),
        'TeamMem',
      )
      if (teamMemEntry) {
        // ...添加到result
      }
    }
    
    // 遥测记录
    if (!hasLoggedInitialLoad) {
      logEvent('tengu_claudemd__initial_load', {
        file_count: result.length,
        user_count: typeCounts['User'] ?? 0,
        project_count: typeCounts['Project'] ?? 0,
        // ...
      })
    }
    
    return result
  },
)
```

**加载顺序完整链：**
```
Managed → Managed rules → User → User rules → Project (从根到CWD) → Local → AutoMem → TeamMem
```

---

## 4. Memory内容组装

### 4.1 context.ts 中getUserContext()的memory部分

**文件路径:** `context.ts`

```typescript
export const getUserContext = memoize(
  async (): Promise<{ [k: string]: string }> => {
    const startTime = Date.now()
    
    // 禁用检查
    const shouldDisableClaudeMd =
      isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_CLAUDE_MDS) ||
      (isBareMode() && getAdditionalDirectoriesForClaudeMd().length === 0)
    
    // 获取记忆文件
    const claudeMd = shouldDisableClaudeMd
      ? null
      : getClaudeMds(filterInjectedMemoryFiles(await getMemoryFiles()))
    
    // 缓存内容供其他模块使用
    setCachedClaudeMdContent(claudeMd || null)
    
    return {
      ...(claudeMd && { claudeMd }),
      currentDate: `Today's date is ${getLocalISODate()}.`,
    }
  },
)
```

**关键点：**
- `filterInjectedMemoryFiles()` - 过滤通过attachment注入的记忆
- `getClaudeMds()` - 格式化为提示词文本
- `setCachedClaudeMdContent()` - 缓存供yoloClassifier等使用

### 4.2 系统提示词中的memory注入

**注入位置：** User Context区块

**格式化输出（getClaudeMds函数）：**

```typescript
export const getClaudeMds = (
  memoryFiles: MemoryFileInfo[],
  filter?: (type: MemoryType) => boolean,
): string => {
  const memories: string[] = []
  
  for (const file of memoryFiles) {
    if (filter && !filter(file.type)) continue
    
    const description =
      file.type === 'Project'
        ? ' (project instructions, checked into the codebase)'
        : file.type === 'Local'
          ? " (user's private project instructions, not checked in)"
          : file.type === 'TeamMem'
            ? ' (shared team memory, synced across the organization)'
            : file.type === 'AutoMem'
              ? " (user's auto-memory, persists across conversations)"
              : " (user's private global instructions)"
    
    if (feature('TEAMMEM') && file.type === 'TeamMem') {
      memories.push(
        `Contents of ${file.path}${description}:\n\n` +
        `<team-memory-content source="shared">\n${content}\n</team-memory-content>`,
      )
    } else {
      memories.push(`Contents of ${file.path}${description}:\n\n${content}`)
    }
  }
  
  if (memories.length === 0) return ''
  
  return `${MEMORY_INSTRUCTION_PROMPT}\n\n${memories.join('\n\n')}`
}
```

**输出格式：**
```
Codebase and user instructions are shown below. Be sure to adhere to these instructions.

Contents of /path/to/CLAUDE.md (project instructions, checked into the codebase):

[文件内容]

Contents of ~/.claude/projects/hash/memory/MEMORY.md (user's auto-memory, persists across conversations):

[记忆内容]
```

---

## 5. 团队记忆同步

### 5.1 services/teamMemorySync/index.ts

**文件路径:** `services/teamMemorySync/index.ts`

#### SyncState - 同步状态对象（行94-127）

```typescript
export type SyncState = {
  /** 最后已知的服务器checksum（ETag） */
  lastKnownChecksum: string | null
  
  /** 每个key的内容hash（sha256:<hex>） */
  serverChecksums: Map<string, string>
  
  /** 从413响应学习的服务器max_entries限制 */
  serverMaxEntries: number | null
}

export function createSyncState(): SyncState {
  return {
    lastKnownChecksum: null,
    serverChecksums: new Map(),
    serverMaxEntries: null,
  }
}
```

**设计意图：**
- 状态对象化，测试可独立创建
- ETag追踪支持条件请求
- serverChecksums支持delta上传

#### pullTeamMemory() - 拉取团队记忆（行770-867）

```typescript
export async function pullTeamMemory(
  state: SyncState,
  options?: { skipEtagCache?: boolean },
): Promise<{
  success: boolean
  filesWritten: number
  entryCount: number
}> {
  const skipEtagCache = options?.skipEtagCache ?? false

  // OAuth检查
  if (!isUsingOAuth()) {
    return { success: false, filesWritten: 0, entryCount: 0, error: 'OAuth not available' }
  }

  // Git remote检查
  const repoSlug = await getGithubRepo()
  if (!repoSlug) {
    return { success: false, filesWritten: 0, entryCount: 0, error: 'No git remote' }
  }

  // 条件GET请求
  const etag = skipEtagCache ? null : state.lastKnownChecksum
  const result = await fetchTeamMemory(state, repoSlug, etag)
  
  if (result.notModified) {
    return { success: true, filesWritten: 0, entryCount: 0, notModified: true }
  }
  
  if (result.isEmpty || !result.data) {
    state.serverChecksums.clear()
    return { success: true, filesWritten: 0, entryCount: 0 }
  }

  // 更新serverChecksums
  state.serverChecksums.clear()
  if (responseChecksums) {
    for (const [key, hash] of Object.entries(responseChecksums)) {
      state.serverChecksums.set(key, hash)
    }
  }

  // 写入本地文件
  const filesWritten = await writeRemoteEntriesToLocal(entries)
  
  if (filesWritten > 0) {
    clearMemoryFileCaches()  // 清除缓存
  }

  return {
    success: true,
    filesWritten,
    entryCount: Object.keys(entries).length,
  }
}
```

**流程：**
1. OAuth + Git remote前置检查
2. 条件GET（If-None-Match ETag）
3. 更新serverChecksums
4. 并行写入本地文件
5. 清除记忆文件缓存

#### pushTeamMemory() - 推送团队记忆（行889-1099）

```typescript
export async function pushTeamMemory(
  state: SyncState,
): Promise<TeamMemorySyncPushResult> {
  // 前置检查
  if (!isUsingOAuth()) {
    return { success: false, filesUploaded: 0, errorType: 'no_oauth' }
  }
  
  const repoSlug = await getGithubRepo()
  if (!repoSlug) {
    return { success: false, filesUploaded: 0, errorType: 'no_repo' }
  }

  // 读取本地文件（含密钥扫描）
  const localRead = await readLocalTeamMemory(state.serverMaxEntries)
  const entries = localRead.entries
  const skippedSecrets = localRead.skippedSecrets
  
  // 计算本地hash
  const localHashes = new Map<string, string>()
  for (const [key, content] of Object.entries(entries)) {
    localHashes.set(key, hashContent(content))
  }

  // 冲突重试循环
  for (let conflictAttempt = 0; conflictAttempt <= MAX_CONFLICT_RETRIES; conflictAttempt++) {
    // Delta计算：仅上传hash不同的key
    const delta: Record<string, string> = {}
    for (const [key, localHash] of localHashes) {
      if (state.serverChecksums.get(key) !== localHash) {
        delta[key] = entries[key]!
      }
    }
    
    if (Object.keys(delta).length === 0) {
      return { success: true, filesUploaded: 0 }
    }

    // 分批上传（每批<200KB）
    const batches = batchDeltaByBytes(delta)
    let filesUploaded = 0
    
    for (const batch of batches) {
      result = await uploadTeamMemory(state, repoSlug, batch, state.lastKnownChecksum)
      
      if (!result.success) break
      
      // 更成功批次更新serverChecksums
      for (const key of Object.keys(batch)) {
        state.serverChecksums.set(key, localHashes.get(key)!)
      }
      filesUploaded += Object.keys(batch).length
    }
    
    if (result.success) {
      return { success: true, filesUploaded, ...(skippedSecrets.length > 0 && { skippedSecrets }) }
    }
    
    if (!result.conflict) {
      // 学习服务器max_entries
      if (result.serverMaxEntries !== undefined) {
        state.serverMaxEntries = result.serverMaxEntries
      }
      return { success: false, filesUploaded, errorType: result.errorType }
    }
    
    // 412冲突：刷新serverChecksums并重试
    const hashesResult = await fetchTeamMemoryHashes(state, repoSlug)
    if (hashesResult.success && hashesResult.entryChecksums) {
      state.serverChecksums.clear()
      for (const [key, hash] of Object.entries(hashesResult.entryChecksums)) {
        state.serverChecksums.set(key, hash)
      }
    }
  }
  
  return { success: false, errorType: 'conflict' }
}
```

**关键机制：**
- **Delta上传** - 仅上传hash不同的文件
- **分批上传** - 每批<200KB避免gateway限制
- **乐观锁** - If-Match ETag条件PUT
- **冲突重试** - 412时刷新checksums重新计算delta
- **本地优先** - 冲突时本地版本覆盖服务器

---

### 5.2 secretScanner.ts 安全扫描

**文件路径:** `services/teamMemorySync/secretScanner.ts`

#### 密钥规则集（行48-224）

```typescript
const SECRET_RULES: SecretRule[] = [
  // 云服务商
  { id: 'aws-access-token', source: '\\b((?:A3T[A-Z0-9]|AKIA|ASIA)[A-Z2-7]{16})\\b' },
  { id: 'gcp-api-key', source: '\\b(AIza[\\w-]{35})(?:[\\x60\'"\\s;]|$)' },
  { id: 'azure-ad-client-secret', source: '([a-zA-Z0-9_~.]{3}\\dQ~[a-zA-Z0-9_~.-]{31,34})' },
  
  // AI APIs
  { id: 'anthropic-api-key', source: `\\b(${ANT_KEY_PFX}03-[a-zA-Z0-9_\\-]{93}AA)` },
  { id: 'openai-api-key', source: '\\b(sk-(?:proj|svcacct|admin)-...T3BlbkFJ...)' },
  
  // 版本控制
  { id: 'github-pat', source: 'ghp_[0-9a-zA-Z]{36}' },
  { id: 'gitlab-pat', source: 'glpat-[\\w-]{20}' },
  
  // 通讯工具
  { id: 'slack-bot-token', source: 'xoxb-[0-9]{10,13}-[0-9]{10,13}[a-zA-Z0-9-]*' },
  
  // 开发工具
  { id: 'npm-access-token', source: '\\b(npm_[a-zA-Z0-9]{36})' },
  { id: 'pypi-upload-token', source: 'pypi-AgEIcHlwaS5vcmc[\\w-]{50,1000}' },
  
  // 私钥
  { id: 'private-key', source: '-----BEGIN[ A-Z0-9_-]{0,100}PRIVATE KEY-----...', flags: 'i' },
]
```

**规则来源：** gitleaks高置信度规则子集

#### scanForSecrets() - 扫描函数（行277-295）

```typescript
export function scanForSecrets(content: string): SecretMatch[] {
  const matches: SecretMatch[] = []
  const seen = new Set<string>()
  
  for (const rule of getCompiledRules()) {
    if (seen.has(rule.id)) continue
    
    if (rule.re.test(content)) {
      seen.add(rule.id)
      matches.push({
        ruleId: rule.id,
        label: ruleIdToLabel(rule.id),  // "github-pat" → "GitHub PAT"
      })
    }
  }
  
  return matches
}
```

**关键设计：**
- **不返回匹配值** - 仅返回规则ID和标签，避免日志泄露密钥
- **去重** - 每个规则最多报告一次

---

### 5.3 上传/下载流程

**上传流程：**
```
本地文件 → 密钥扫描 → skip含密钥文件 → 计算hash → delta(与serverChecksums比较)
→ 分批(<200KB) → 条件PUT(If-Match ETag) → 更新serverChecksums
→ 冲突(412)? → 刷新hashes → 重算delta → 重试
```

**下载流程：**
```
条件GET(If-None-Match ETag) → 304? → 无变更
→ 404? → 清空serverChecksums
→ 200? → 更新serverChecksums → 并行写入本地 → 清除缓存
```

---

## 6. 从零实现指南

### 6.1 如何设计记忆系统

**核心架构决策：**

1. **存储层：**
   - 文件系统持久化（而非数据库）
   - 分层目录结构（私有/团队）
   - 索引文件 + 独立内容文件

2. **类型系统：**
   - 封闭类型集合（避免膨胀）
   - 每类型有明确用途和scope
   - frontmatter元数据（name/description/type）

3. **检索层：**
   - 模型驱动的相关性选择
   - 文件头扫描（description字段）
   - 年龄追踪和陈旧警告

4. **同步层：**
   - 条件请求（ETag）
   - Delta上传（仅变更文件）
   - 冲突检测和重试

### 6.2 年龄衰减算法

**实现要点：**

```typescript
// 1. 天数计算
function memoryAgeDays(mtimeMs: number): number {
  return Math.max(0, Math.floor((Date.now() - mtimeMs) / 86_400_000))
}

// 2. 人类可读格式
function memoryAge(mtimeMs: number): string {
  const d = memoryAgeDays(mtimeMs)
  if (d === 0) return 'today'
  if (d === 1) return 'yesterday'
  return `${d} days ago`
}

// 3. 陈旧警告阈值
function memoryFreshnessText(mtimeMs: number): string {
  const d = memoryAgeDays(mtimeMs)
  if (d <= 1) return ''  // 今天/昨天无警告
  
  return `This memory is ${d} days old. Verify against current state before asserting as fact.`
}
```

**设计原则：**
- 阈值设为2天（超过昨天触发警告）
- 不自动删除陈旧记忆（用户手动清理）
- 文件引用类记忆警告更严格

### 6.3 团队共享机制

**实现步骤：**

1. **状态管理：**
   ```typescript
   type SyncState = {
     lastKnownChecksum: string | null    // ETag
     serverChecksums: Map<string, string> // sha256:<hex>
     serverMaxEntries: number | null      // 从413学习
   }
   ```

2. **条件请求：**
   ```typescript
   // GET
   headers['If-None-Match'] = `"${etag}"`  // 304 = 无变更
   
   // PUT
   headers['If-Match'] = `"${checksum}"`   // 412 = 冲突
   ```

3. **Delta计算：**
   ```typescript
   const delta = {}
   for (const [key, localHash] of localHashes) {
     if (serverChecksums.get(key) !== localHash) {
       delta[key] = entries[key]
     }
   }
   ```

4. **冲突处理：**
   ```typescript
   if (result.conflict) {
     // 刷新serverChecksums（不含body的probe）
     const hashes = await fetchTeamMemoryHashes()
     // 重算delta（自然排除队友已push的内容）
     // 重试
   }
   ```

5. **安全扫描：**
   ```typescript
   for (const file of files) {
     const matches = scanForSecrets(content)
     if (matches.length > 0) {
       skippedSecrets.push({ path, ruleId: matches[0].ruleId })
       continue  // 跳过含密钥文件
     }
     entries[path] = content
   }
   ```

---

## 附录：关键文件索引

| 文件路径 | 核心职责 |
|---------|---------|
| `memdir/memdir.ts` | 提示词构建、内容截断、目录创建 |
| `memdir/paths.ts` | 路径解析、功能开关、安全验证 |
| `memdir/memoryTypes.ts` | 类型定义、提示词常量 |
| `memdir/memoryAge.ts` | 年龄计算、陈旧警告 |
| `memdir/findRelevantMemories.ts` | 相关性查找、模型选择 |
| `memdir/memoryScan.ts` | 文件扫描、清单格式化 |
| `memdir/teamMemPaths.ts` | 团队路径、symlink安全 |
| `memdir/teamMemPrompts.ts` | 组合提示词生成 |
| `utils/claudemd.ts` | CLAUDE.md加载、@include处理 |
| `context.ts` | 上下文组装、缓存管理 |
| `services/teamMemorySync/index.ts` | 同步核心、pull/push逻辑 |
| `services/teamMemorySync/secretScanner.ts` | 密钥扫描 |
| `services/teamMemorySync/watcher.ts` | 文件监听、自动同步 |

---

*Memory系统分析完成*