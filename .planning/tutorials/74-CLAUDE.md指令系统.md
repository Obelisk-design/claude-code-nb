# CLAUDE.md 指令系统详解

**分析日期:** 2026-04-02  
**源码位置:** `utils/claudemd.ts`

---

## 一、CLAUDE.md 概述

### 1.1 什么是 CLAUDE.md

CLAUDE.md 是 Claude Code CLI 的**指令注入系统**，允许用户通过 Markdown 文件向 AI 提供项目特定的行为规范、编码风格、工作流程等指导信息。这些指令会在每次对话开始时自动加载到 AI 的上下文中，影响其决策和行为。

### 1.2 核心功能

- **多层级指令管理**：支持全局、用户级、项目级、本地级四层指令体系
- **@include 指令**：支持文件引用，实现指令模块化
- **条件规则**：通过 frontmatter 的 `paths` 字段实现按路径匹配的动态加载
- **HTML 注释过滤**：自动剥离块级 HTML 注释，避免作者笔记干扰 AI
- **自动截断**：对 MEMORY.md 入口点实施字符和行数限制

### 1.3 文件类型定义

```typescript
// 源码位置: utils/memory/types.ts
export const MEMORY_TYPE_VALUES = [
  'User',      // 用户全局指令 (~/.claude/CLAUDE.md)
  'Project',   // 项目指令 (CLAUDE.md, .claude/CLAUDE.md)
  'Local',     // 本地私有指令 (CLAUDE.local.md)
  'Managed',   // 管理员指令 (/etc/claude-code/CLAUDE.md)
  'AutoMem',   // 自动记忆入口点 (MEMORY.md)
  'TeamMem',   // 团队记忆入口点 (团队共享)
] as const

export type MemoryType = (typeof MEMORY_TYPE_VALUES)[number]
```

### 1.4 MemoryFileInfo 结构

```typescript
// 源码位置: utils/claudemd.ts:229-243
export type MemoryFileInfo = {
  path: string                    // 文件绝对路径
  type: MemoryType                // 指令类型
  content: string                 // 处理后的内容
  parent?: string                 // 引用此文件的父文件路径
  globs?: string[]                // 条件规则的 glob 匹配模式
  contentDiffersFromDisk?: boolean // 内容是否经过变换
  rawContent?: string             // 原始磁盘内容（用于缓存）
}
```

---

## 二、utils/claudemd.ts 核心实现

### 2.1 文件头部注释（关键设计文档）

文件开头 1-26 行是系统设计的核心说明：

```typescript
/**
 * Files are loaded in the following order:
 *
 * 1. Managed memory (eg. /etc/claude-code/CLAUDE.md) - Global instructions for all users
 * 2. User memory (~/.claude/CLAUDE.md) - Private global instructions for all projects
 * 3. Project memory (CLAUDE.md, .claude/CLAUDE.md, and .claude/rules/*.md in project roots) 
 *    - Instructions checked into the codebase
 * 4. Local memory (CLAUDE.local.md in project roots) - Private project-specific instructions
 *
 * Files are loaded in reverse order of priority, i.e. the latest files are highest priority
 * with the model paying more attention to them.
 *
 * File discovery:
 * - User memory is loaded from the user's home directory
 * - Project and Local files are discovered by traversing from the current directory up to root
 * - Files closer to the current directory have higher priority (loaded later)
 * - CLAUDE.md, .claude/CLAUDE.md, and all .md files in .claude/rules/ are checked in each directory
 *
 * Memory @include directive:
 * - Memory files can include other files using @ notation
 * - Syntax: @path, @./relative/path, @~/home/path, or @/absolute/path
 * - @path (without prefix) is treated as a relative path (same as @./path)
 * - Works in leaf text nodes only (not inside code blocks or code strings)
 * - Included files are added as separate entries before the including file
 * - Circular references are prevented by tracking processed files
 * - Non-existent files are silently ignored
 */
```

**设计要点解读：**

1. **优先级反向加载**：后加载的文件优先级更高，AI 更关注它们
2. **向上遍历发现**：从 CWD 向上遍历到根目录，越靠近 CWD 的文件优先级越高
3. **@include 限制**：只在叶子文本节点生效，代码块内不解析

### 2.2 支持的文本文件扩展名

```typescript
// 源码位置: utils/claudemd.ts:96-227
const TEXT_FILE_EXTENSIONS = new Set([
  // Markdown and text
  '.md', '.txt', '.text',
  // Data formats
  '.json', '.yaml', '.yml', '.toml', '.xml', '.csv',
  // Web
  '.html', '.htm', '.css', '.scss', '.sass', '.less',
  // JavaScript/TypeScript
  '.js', '.ts', '.tsx', '.jsx', '.mjs', '.cjs', '.mts', '.cts',
  // Python
  '.py', '.pyi', '.pyw',
  // Ruby, Go, Rust, Java/Kotlin/Scala, C/C++, C#, Swift
  // Shell scripts (.sh, .bash, .zsh, .fish, .ps1, .bat, .cmd)
  // Config files (.env, .ini, .cfg, .conf, .config, .properties)
  // Database (.sql, '.graphql', '.gql')
  // Protocol (.proto)
  // Frontend frameworks (.vue, '.svelte', '.astro')
  // Templating (.ejs, '.hbs', '.pug', '.jade')
  // Other languages (.php, .pl, .pm, '.lua', '.r', '.R', '.dart', ...)
  // Build files (.cmake, '.make', '.makefile', '.gradle', '.sbt')
  // Documentation (.rst, '.adoc', '.asciidoc', '.org', '.tex', '.latex')
  // Lock files (.lock)
  // Misc (.log, '.diff', '.patch')
])
```

**设计意图**：防止二进制文件（图片、PDF 等）被错误加载到内存中。

### 2.3 getMemoryFiles 主加载函数

```typescript
// 源码位置: utils/claudemd.ts:790-1075
export const getMemoryFiles = memoize(
  async (forceIncludeExternal: boolean = false): Promise<MemoryFileInfo[]> => {
    const result: MemoryFileInfo[] = []
    const processedPaths = new Set<string>()
    
    // 1. 加载 Managed 文件（管理员策略）
    const managedClaudeMd = getMemoryPath('Managed')
    result.push(...(await processMemoryFile(managedClaudeMd, 'Managed', ...)))
    
    // 2. 加载 Managed .claude/rules/*.md
    result.push(...(await processMdRules({
      rulesDir: getManagedClaudeRulesDir(),
      type: 'Managed',
      conditionalRule: false,
    })))
    
    // 3. 加载 User 文件（如果 userSettings 启用）
    if (isSettingSourceEnabled('userSettings')) {
      result.push(...(await processMemoryFile(getMemoryPath('User'), 'User', ...)))
      result.push(...(await processMdRules({
        rulesDir: getUserClaudeRulesDir(),
        type: 'User',
        includeExternal: true,
      })))
    }
    
    // 4. 从 CWD 向上遍历加载 Project 和 Local 文件
    const dirs: string[] = []
    let currentDir = getOriginalCwd()
    while (currentDir !== parse(currentDir).root) {
      dirs.push(currentDir)
      currentDir = dirname(currentDir)
    }
    
    // 从根向下处理（后处理的优先级更高）
    for (const dir of dirs.reverse()) {
      // CLAUDE.md (Project)
      // .claude/CLAUDE.md (Project)
      // .claude/rules/*.md (Project)
      // CLAUDE.local.md (Local)
    }
    
    // 5. 加载 AutoMem 入口点（MEMORY.md）
    if (isAutoMemoryEnabled()) {
      const { info: memdirEntry } = await safelyReadMemoryFileAsync(
        getAutoMemEntrypoint(), 'AutoMem'
      )
      if (memdirEntry) result.push(memdirEntry)
    }
    
    // 6. 加载 TeamMem 入口点（团队共享）
    if (feature('TEAMMEM') && teamMemPaths!.isTeamMemoryEnabled()) {
      const { info: teamMemEntry } = await safelyReadMemoryFileAsync(
        teamMemPaths!.getTeamMemEntrypoint(), 'TeamMem'
      )
      if (teamMemEntry) result.push(teamMemEntry)
    }
    
    return result
  }
)
```

### 2.4 processMemoryFile 文件处理

```typescript
// 源码位置: utils/claudemd.ts:618-685
export async function processMemoryFile(
  filePath: string,
  type: MemoryType,
  processedPaths: Set<string>,
  includeExternal: boolean,
  depth: number = 0,
  parent?: string,
): Promise<MemoryFileInfo[]> {
  // 路径标准化（处理 Windows 驱动器字母大小写）
  const normalizedPath = normalizePathForComparison(filePath)
  
  // 循环引用检测 & 最大深度限制
  if (processedPaths.has(normalizedPath) || depth >= MAX_INCLUDE_DEPTH) {
    return []
  }
  
  // 排除检测（claudeMdExcludes 设置）
  if (isClaudeMdExcluded(filePath, type)) {
    return []
  }
  
  // 符号链接解析
  const { resolvedPath, isSymlink } = safeResolvePath(getFsImplementation(), filePath)
  
  // 读取并解析文件
  const { info: memoryFile, includePaths } = await safelyReadMemoryFileAsync(
    filePath, type, resolvedPath
  )
  
  // 处理 @include 引用
  for (const resolvedIncludePath of includePaths) {
    const includedFiles = await processMemoryFile(
      resolvedIncludePath, type, processedPaths, includeExternal,
      depth + 1, filePath
    )
    result.push(...includedFiles)
  }
  
  return result
}
```

### 2.5 getClaudeMds 格式化输出

```typescript
// 源码位置: utils/claudemd.ts:1153-1195
export const getClaudeMds = (
  memoryFiles: MemoryFileInfo[],
  filter?: (type: MemoryType) => boolean,
): string => {
  const memories: string[] = []
  
  for (const file of memoryFiles) {
    if (file.content) {
      const description =
        file.type === 'Project'
          ? ' (project instructions, checked into the codebase)'
          : file.type === 'Local'
            ? " (user's private project instructions, not checked in)"
            : file.type === 'TeamMem'
              ? ' (shared team memory, synced across the organization)'
              : file.type === 'AutoMem'
                ? " (user's auto-memory, persists across conversations)"
                : " (user's private global instructions for all projects)"
      
      memories.push(`Contents of ${file.path}${description}:\n\n${file.content}`)
    }
  }
  
  if (memories.length === 0) return ''
  
  return `${MEMORY_INSTRUCTION_PROMPT}\n\n${memories.join('\n\n')}`
}
```

**输出示例格式**：

```
Codebase and user instructions are shown below. Be sure to adhere to these instructions. 
IMPORTANT: These instructions OVERRIDE default default behavior and MUST be followed exactly as written.

Contents of /etc/claude-code/CLAUDE.md (managed policy):
...

Contents of ~/.claude/CLAUDE.md (user's private global instructions for all projects):
...

Contents of /project/CLAUDE.md (project instructions, checked into the codebase):
...
```

---

## 三、@include 指令详解

### 3.1 指令语法

@include 指令允许在 CLAUDE.md 文件中引用其他文件：

```markdown
@include 共享配置 @./shared/style-guide.md
@include 用户目录 @~/configs/typescript-rules.md
@include 绝对路径 @/etc/claude-code/shared-rules.md
```

**四种路径格式：**

| 格式 | 示例 | 解析规则 |
|------|------|----------|
| 无前缀 | `@path/to/file.md` | 等同于 `@./path/to/file.md` |
| 相对路径 | `@./relative/path.md` | 相对于当前 CLAUDE.md 所在目录 |
| 用户目录 | `@~/home/path.md` | 相对于 `~/.claude/` 目录 |
| 绝对路径 | `@/absolute/path.md` | 绝对路径 |

### 3.2 正则解析实现

```typescript
// 源码位置: utils/claudemd.ts:451-491
function extractPathsFromText(textContent: string) {
  // 正则：匹配行首或空白后的 @路径
  const includeRegex = /(?:^|\s)@((?:[^\s\\]|\\ )+)/g
  
  let match
  while ((match = includeRegex.exec(textContent)) !== null) {
    let path = match[1]
    if (!path) continue
    
    // 剔除片段标识符 (#heading)
    const hashIndex = path.indexOf('#')
    if (hashIndex !== -1) {
      path = path.substring(0, hashIndex)
    }
    
    // 反转义空格（支持 @path/to/file\ with\ spaces.md）
    path = path.replace(/\\ /g, ' ')
    
    // 验证路径有效性
    const isValidPath =
      path.startsWith('./') ||
      path.startsWith('~/') ||
      (path.startsWith('/') && path !== '/') ||
      (!path.startsWith('@') &&
        !path.match(/^[#%^&*()]+/) &&
        path.match(/^[a-zA-Z0-9._-]/))
    
    if (isValidPath) {
      const resolvedPath = expandPath(path, dirname(basePath))
      absolutePaths.add(resolvedPath)
    }
  }
}
```

### 3.3 Markdown Token 递归解析

@include 解析使用 marked 的 Lexer 进行 Markdown 解析，递归遍历 token 树：

```typescript
// 源码位置: utils/claudemd.ts:494-534
function processElements(elements: MarkdownToken[]) {
  for (const element of elements) {
    // 跳过代码块和代码片段（@include 在代码块内无效）
    if (element.type === 'code' || element.type === 'codespan') {
      continue
    }
    
    // HTML token 特殊处理：剥离注释后检查残留内容
    if (element.type === 'html') {
      const trimmed = raw.trimStart()
      if (trimmed.startsWith('<!--') && trimmed.includes('-->')) {
        const residue = raw.replace(/<!--[\s\S]*?-->/g, '')
        if (residue.trim().length > 0) {
          extractPathsFromText(residue)
        }
      }
      continue
    }
    
    // 处理文本节点
    if (element.type === 'text') {
      extractPathsFromText(element.text || '')
    }
    
    // 递归子 token
    if (element.tokens) processElements(element.tokens)
    if (element.items) processElements(element.items)
  }
}
```

**关键设计：**

- `code` 和 `codespan` token 被跳过 → 代码块内的 `@include` 不生效
- HTML 注释内的 `@include` 被跳过 → 作者笔记不影响 AI
- 递归处理嵌套结构 → 支持列表、引用等复杂 Markdown

### 3.4 最大深度限制

```typescript
// 源码位置: utils/claudemd.ts:537
const MAX_INCLUDE_DEPTH = 5
```

防止无限递归的循环引用。深度超过 5 层的 @include 链会被截断。

### 3.5 外部文件引用控制

```typescript
// 源码位置: utils/claudemd.ts:667-670
for (const resolvedIncludePath of resolvedIncludePaths) {
  const isExternal = !pathInOriginalCwd(resolvedIncludePath)
  if (isExternal && !includeExternal) {
    continue  // 跳过外部文件
  }
}
```

外部文件（不在 CWD 范围内）需要用户显式批准才会被加载。

---

## 四、加载优先级详解

### 4.1 四层优先级体系

| 层级 | 类型 | 文件位置 | 特点 |
|------|------|----------|------|
| 1 (最低) | Managed | `/etc/claude-code/CLAUDE.md` | 系统管理员强制策略 |
| 2 | User | `~/.claude/CLAUDE.md` | 用户全局偏好 |
| 3 | Project | `CLAUDE.md`, `.claude/CLAUDE.md` | 项目团队规范 |
| 4 (最高) | Local | `CLAUDE.local.md` | 用户私有项目配置 |

### 4.2 反向优先级原理

```typescript
// 源码注释原文:
// "Files are loaded in reverse order of priority, i.e. the latest files are 
//  highest priority with the model paying more attention to them."
```

**AI 注意力机制**：后加载的内容出现在 prompt 尾部，AI 更关注尾部内容（近因效应）。

### 4.3 目录遍历顺序

```typescript
// 源码位置: utils/claudemd.ts:850-878
const dirs: string[] = []
let currentDir = originalCwd

// 向上收集目录
while (currentDir !== parse(currentDir).root) {
  dirs.push(currentDir)
  currentDir = dirname(currentDir)
}

// 反向遍历（从根到 CWD）
for (const dir of dirs.reverse()) {
  // 处理该目录的 Project 和 Local 文件
}
```

**示例**：假设 CWD 是 `/home/user/project/src/components`

遍历顺序：
1. `/home` (根目录级，最低优先级)
2. `/home/user`
3. `/home/user/project`
4. `/home/user/project/src`
5. `/home/user/project/src/components` (CWD，最高优先级)

### 4.4 同目录内的加载顺序

每个目录内按此顺序加载：

```typescript
// 源码位置: utils/claudemd.ts:886-933
for (const dir of dirs.reverse()) {
  // 1. CLAUDE.md (Project)
  const projectPath = join(dir, 'CLAUDE.md')
  
  // 2. .claude/CLAUDE.md (Project)
  const dotClaudePath = join(dir, '.claude', 'CLAUDE.md')
  
  // 3. .claude/rules/*.md (Project, unconditional)
  const rulesDir = join(dir, '.claude', 'rules')
  
  // 4. CLAUDE.local.md (Local)
  const localPath = join(dir, 'CLAUDE.local.md')
}
```

### 4.5 Worktree 嵌套处理

```typescript
// 源码位置: utils/claudemd.ts:867-884
// 当在 git worktree 中运行时，避免重复加载主仓库的文件
const gitRoot = findGitRoot(originalCwd)
const canonicalRoot = findCanonicalGitRoot(originalCwd)
const isNestedWorktree =
  gitRoot !== null &&
  canonicalRoot !== null &&
  normalizePathForComparison(gitRoot) !== normalizePathForComparison(canonicalRoot)

// 跳过主仓库中 worktree 外的 Project 文件
const skipProject =
  isNestedWorktree &&
  pathInWorkingPath(dir, canonicalRoot) &&
  !pathInWorkingPath(dir, gitRoot)
```

---

## 五、条件规则（Conditional Rules）

### 5.1 Frontmatter paths 配置

```yaml
---
paths: "src/**/*.ts, src/**/*.tsx"
---

# TypeScript 文件专用规则
- 使用严格的类型检查
- 所有函数必须有返回类型声明
```

**paths 字段支持：**

- 单个 glob 模式：`paths: "src/*.ts"`
- 多个模式（逗号分隔）：`paths: "src/*.ts, src/*.tsx"`
- YAML 数组：`paths: ["src/*.ts", "src/*.tsx"]`
- Brace 展开：`paths: "src/*.{ts,tsx}"` → 展开为 `src/*.ts` 和 `src/*.tsx`

### 5.2 Frontmatter 解析实现

```typescript
// 源码位置: utils/frontmatterParser.ts:189-232
export function splitPathInFrontmatter(input: string | string[]): string[] {
  if (Array.isArray(input)) {
    return input.flatMap(splitPathInFrontmatter)
  }
  
  // 逗号分隔（尊重 brace 层级）
  const parts: string[] = []
  let current = ''
  let braceDepth = 0
  
  for (const char of input) {
    if (char === '{') braceDepth++
    else if (char === '}') braceDepth--
    else if (char === ',' && braceDepth === 0) {
      parts.push(current.trim())
      current = ''
    } else {
      current += char
    }
  }
  
  // 展开 brace 模式
  return parts.flatMap(pattern => expandBraces(pattern))
}

function expandBraces(pattern: string): string[] {
  // src/*.{ts,tsx} → ["src/*.ts", "src/*.tsx"]
  // {a,b}/{c,d} → ["a/c", "a/d", "b/c", "b/d"]
}
```

### 5.3 条件规则匹配流程

```typescript
// 源码位置: utils/claudemd.ts:1354-1397
export async function processConditionedMdRules(
  targetPath: string,   // 目标文件路径
  rulesDir: string,     // .claude/rules/ 目录
  type: MemoryType,
  processedPaths: Set<string>,
  includeExternal: boolean,
): Promise<MemoryFileInfo[]> {
  // 1. 获取所有带 paths frontmatter 的规则文件
  const conditionedRuleMdFiles = await processMdRules({
    rulesDir, type, processedPaths, includeExternal,
    conditionalRule: true,  // 只返回有 globs 的文件
  })
  
  // 2. 过滤匹配目标路径的规则
  return conditionedRuleMdFiles.filter(file => {
    if (!file.globs || file.globs.length === 0) return false
    
    // 计算相对路径
    const baseDir = type === 'Project'
      ? dirname(dirname(rulesDir))  // .claude 的父目录
      : getOriginalCwd()            // Managed/User 规则相对 CWD
    
    const relativePath = relative(baseDir, targetPath)
    
    // 使用 ignore 库匹配 glob
    return ignore().add(file.globs).ignores(relativePath)
  })
}
```

### 5.4 条件规则加载时机

条件规则在以下场景加载：

1. **Read 文件时**：通过 `getNestedMemoryAttachmentsForReadPath()` 加载
2. **Edit/Write 文件时**：匹配目标文件路径的条件规则被注入
3. **嵌套目录遍历**：`getMemoryFilesForNestedDirectory()` 处理目录间的条件规则

---

## 六、HTML 注释剥离

### 6.1 设计目的

允许作者在 CLAUDE.md 中添加开发笔记而不影响 AI：

```markdown
<!-- 这是作者的内部备注，AI 不会看到 -->
# 项目规范

<!-- TODO: 需要更新这部分 -->
- 使用 TypeScript strict 模式
```

### 6.2 实现机制

```typescript
// 源码位置: utils/claudemd.ts:292-334
export function stripHtmlComments(content: string): {
  content: string
  stripped: boolean
} {
  if (!content.includes('<!--')) {
    return { content, stripped: false }
  }
  
  return stripHtmlCommentsFromTokens(new Lexer({ gfm: false }).lex(content))
}

function stripHtmlCommentsFromTokens(tokens): { content: string; stripped: boolean } {
  const commentSpan = /<!--[\s\S]*?-->/g
  
  for (const token of tokens) {
    if (token.type === 'html') {
      const trimmed = token.raw.trimStart()
      if (trimmed.startsWith('<!--') && trimmed.includes('-->')) {
        // 剥离注释，保留残留内容
        const residue = token.raw.replace(commentSpan, '')
        if (residue.trim().length > 0) {
          result += residue  // 如 <!-- note --> 后的文字
        }
        continue
      }
    }
    result += token.raw
  }
  
  return { content: result, stripped: true }
}
```

**关键特性：**

- 只剥离**块级** HTML 注释（独立的 HTML token）
- 行内注释（段落内的 `<!-- -->`）保留
- 代码块内的注释保留（code token 被跳过）
- 未闭合的注释保留（避免误删内容）

---

## 七、排除与安全机制

### 7.1 claudeMdExcludes 设置

```typescript
// 源码位置: utils/claudemd.ts:547-573
function isClaudeMdExcluded(filePath: string, type: MemoryType): boolean {
  // 只对 User、Project、Local 类型生效
  if (type !== 'User' && type !== 'Project' && type !== 'Local') {
    return false
  }
  
  const patterns = getInitialSettings().claudeMdExcludes
  if (!patterns || patterns.length === 0) return false
  
  // 使用 picomatch 匹配
  return picomatch.isMatch(normalizedPath, expandedPatterns, { dot: true })
}
```

**配置示例**（settings.json）：

```json
{
  "claudeMdExcludes": [
    "/tmp/project/CLAUDE.md",
    "**/legacy/**/*.md"
  ]
}
```

### 7.2 符号链接处理

```typescript
// 源码位置: utils/claudemd.ts:639-648
const { resolvedPath, isSymlink } = safeResolvePath(getFsImplementation(), filePath)

processedPaths.add(normalizedPath)
if (isSymlink) {
  // 同时记录解析后的真实路径，防止通过符号链接绕过循环检测
  processedPaths.add(normalizePathForComparison(resolvedPath))
}
```

### 7.3 排除模式符号链接解析

```typescript
// 源码位置: utils/claudemd.ts:581-612
function resolveExcludePatterns(patterns: string[]): string[] {
  // 对绝对路径模式，解析符号链接前缀
  // macOS: /tmp → /private/tmp
  for (const normalized of patterns) {
    if (!normalized.startsWith('/')) continue
    
    // 找到静态前缀
    const globStart = normalized.search(/[*?{[]/)
    const staticPrefix = globStart === -1 ? normalized : normalized.slice(0, globStart)
    
    try {
      const resolvedDir = fs.realpathSync(dirname(staticPrefix))
      if (resolvedDir !== dirToResolve) {
        expanded.push(resolvedDir + normalized.slice(dirToResolve.length))
      }
    } catch { /* 目录不存在 */ }
  }
  
  return expanded
}
```

---

## 八、最佳实践

### 8.1 文件组织结构

推荐的项目指令组织结构：

```
project-root/
├── CLAUDE.md                 # 主项目指令（提交到 git）
├── CLAUDE.local.md           # 用户私有配置（不提交）
├── .claude/
│   ├── CLAUDE.md             # 补充项目指令
│   └── rules/
│       ├── typescript.md     # TypeScript 专用规则
│       ├── react.md          # React 专用规则
│       ├── api.md            # API 开发规则
│       └── testing.md        # 测试规则
│       └── backend/
│           ├── database.md   # 后端子目录规则
│           └── auth.md       # 认证规则
└── .claudeignore             # 排除配置
```

### 8.2 条件规则示例

```markdown
---
paths: "src/components/**/*.tsx"
---

# React 组件开发规范

1. 所有组件必须使用 TypeScript
2. 组件文件名使用 PascalCase
3. 每个组件必须有对应的测试文件
4. 使用 named exports，避免 default exports
```

```markdown
---
paths: "backend/api/**/*.ts"
---

# API 端点开发规范

1. 所有端点必须有输入验证
2. 使用 Zod 定义 schema
3. 错误响应遵循 RFC 7807 Problem Details
```

### 8.3 @include 模块化

```markdown
# ~/.claude/CLAUDE.md

@include 共享 TypeScript 规则 @./shared/typescript.md
@include 共享 Git 规范 @./shared/git-conventions.md
@include 项目特定配置 @~/projects/work-project/CLAUDE.local.md
```

### 8.4 内容长度控制

```typescript
// 源码位置: utils/claudemd.ts:92
export const MAX_MEMORY_CHARACTER_COUNT = 40000
```

建议每个 CLAUDE.md 文件控制在 40000 字符以内，避免超出 AI 上下文窗口。

### 8.5 HTML 注释使用

```markdown
# 项目编码规范

<!-- 作者备注：以下规则基于团队 2024 年评审结果 -->
- 使用 ESLint + Prettier 进行代码格式化
- 提交前必须通过所有 lint 检查

<!-- TODO: 待补充 Python 规则 -->
## Python 开发

<!-- 暂时跳过 -->
```

---

## 九、用户编写指南

### 9.1 基础 CLAUDE.md 模板

```markdown
# 项目名称 开发规范

## 项目概述
简述项目目的、技术栈和团队结构。

## 编码规范

### 语言特定规则
- TypeScript: strict 模式，显式返回类型
- Python: PEP 8，Black 格式化

### 命名约定
- 变量: camelCase
- 函数: camelCase
- 类: PascalCase
- 文件: kebab-case

## 工作流程

### Git 提交规范
- feat: 新功能
- fix: 修复
- docs: 文档
- refactor: 重构

### PR 要求
- 必须有描述
- 必须关联 Issue
- 必须通过 CI

## 禁止事项
- 不要直接修改生成的文件
- 不要提交 .env 文件
- 不要绕过 lint 检查
```

### 9.2 条件规则编写

```markdown
---
paths: ["src/api/**/*.ts", "src/services/**/*.ts"]
---

# API 层开发规则

## 错误处理
所有 API 函数必须：
1. 返回 `Result<T, ApiError>` 类型
2. 记录错误日志
3. 提供用户友好消息

## 测试要求
- 单元测试覆盖核心逻辑
- 集成测试覆盖端点
- Mock 外部依赖
```

### 9.3 @include 组织

**主 CLAUDE.md：**

```markdown
# 项目规范

@include 共享编码规则 @./shared/coding.md
@include Git 工作流 @./shared/git.md

## 项目特定规则
（仅适用于本项目的内容）
```

**共享文件（./shared/coding.md）：**

```markdown
# 通用编码规范

## 代码风格
- 最大行宽：100 字符
- 缩进：2 空格
- 分号：必须

## 注释规范
- 公共 API 必须有 JSDoc
- 复杂逻辑必须有解释
```

### 9.4 本地私有配置

**CLAUDE.local.md（不提交到 git）：**

```markdown
# 个人开发偏好

## IDE 设置
- 我使用 VS Code
- 自动保存延迟：500ms

## 个人快捷方式
- 偏好使用 `cc` 作为 `claude-code` 别名

## 私有凭证提醒
- API key 在 ~/.config/api-keys.json
- 不要在对话中提及具体 key 值
```

### 9.5 调试指令加载

使用 `/memory` 命令查看当前加载的指令文件：

```
> /memory

Loaded memory files:
  - /etc/claude-code/CLAUDE.md (Managed)
  - ~/.claude/CLAUDE.md (User)
  - ./CLAUDE.md (Project)
  - ./.claude/rules/typescript.md (Project, paths: src/**/*.ts)
  - ./CLAUDE.local.md (Local)
```

### 9.6 常见问题解决

**Q: 指令没有被 AI 遵守？**

检查优先级：
1. 确认文件位置是否正确
2. 确认 settingsSources 是否启用对应层级
3. 确认内容长度未超限
4. 确认没有被 claudeMdExcludes 排除

**Q: 条件规则没有生效？**

检查 paths 配置：
1. glob 模式相对于 .claude 的父目录
2. 使用 `/memory` 验证文件是否加载
3. 确认目标文件路径匹配 glob

**Q: @include 引用失败？**

检查：
1. 路径格式是否正确
2. 文件扩展名是否在允许列表中
3. 是否超出最大深度（5 层）
4. 是否是外部文件且未获批准

---

## 十、源码关键位置索引

| 功能 | 源码位置 | 行号 |
|------|----------|------|
| 文件头部设计文档 | `utils/claudemd.ts` | 1-26 |
| MemoryType 定义 | `utils/memory/types.ts` | 1-13 |
| MemoryFileInfo 结构 | `utils/claudemd.ts` | 229-243 |
| 文本扩展名列表 | `utils/claudemd.ts` | 96-227 |
| getMemoryFiles 主函数 | `utils/claudemd.ts` | 790-1075 |
| processMemoryFile | `utils/claudemd.ts` | 618-685 |
| @include 正则解析 | `utils/claudemd.ts` | 451-491 |
| Token 递归解析 | `utils/claudemd.ts` | 494-534 |
| MAX_INCLUDE_DEPTH | `utils/claudemd.ts` | 537 |
| HTML 注释剥离 | `utils/claudemd.ts` | 292-334 |
| Frontmatter 解析 | `utils/frontmatterParser.ts` | 130-175 |
| paths 分割展开 | `utils/frontmatterParser.ts` | 189-232 |
| 条件规则匹配 | `utils/claudemd.ts` | 1354-1397 |
| 排除机制 | `utils/claudemd.ts` | 547-612 |
| getClaudeMds 输出 | `utils/claudemd.ts` | 1153-1195 |
| Worktree 处理 | `utils/claudemd.ts` | 867-884 |

---

*CLAUDE.md 指令系统分析完成*