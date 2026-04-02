# GrepTool 工具详解

**分析日期:** 2026-04-02

## 1. 概述

### 工具作用

GrepTool 是 Claude Code CLI 中基于 **ripgrep (rg)** 构建的高性能代码搜索工具。它提供了:

- **正则表达式搜索**: 支持完整的正则语法，如 `log.*Error`、`function\s+\w+`
- **多种输出模式**: 文件列表、匹配内容、计数统计
- **智能过滤**: glob 模式、文件类型过滤
- **权限优化**: 自动处理文件系统权限和忽略模式

核心设计理念:
```typescript
// tools/GrepTool/prompt.ts:7-17
'ALWAYS use Grep for search tasks. NEVER invoke `grep` or `rg` as a Bash command.'
'The Grep tool has been optimized for correct permissions and access.'
```

### 代码搜索能力

| 能力 | 说明 |
|------|------|
| 正则搜索 | ripgrep 正则引擎，非标准 grep 语法 |
| 多行匹配 | `multiline: true` 支持跨行模式 |
| 上下文显示 | `-A`/`-B`/`-C` 参数显示匹配前后行 |
| 分页控制 | `head_limit` + `offset` 支持结果分页 |
| 大小写控制 | `-i` 参数切换大小写敏感 |

---

## 2. 输入 Schema

### 核心参数

工具定义位于 `tools/GrepTool/GrepTool.ts:33-89`:

```typescript
const inputSchema = lazySchema(() =>
  z.strictObject({
    pattern: z.string().describe(
      'The regular expression pattern to search for in file contents'
    ),
    path: z.string().optional().describe(
      'File or directory to search in. Defaults to current working directory.'
    ),
    glob: z.string().optional().describe(
      'Glob pattern to filter files (e.g. "*.js", "*.{ts,tsx}")'
    ),
    output_mode: z.enum(['content', 'files_with_matches', 'count'])
      .optional().describe(
        'Output mode: "content" shows matching lines, "files_with_matches" shows file paths (default), "count" shows match counts'
      ),
    // ... 更多参数
  })
)
```

### pattern 参数

**必填参数**，定义搜索的正则表达式模式:

```typescript
// tools/GrepTool/GrepTool.ts:35-39
pattern: z.string().describe(
  'The regular expression pattern to search for in file contents'
)
```

**关键注意事项**:
- 使用 **ripgrep 语法**，非标准 grep
- 花括号需要转义: `interface\{\}` 搜索 Go 代码中的 `interface{}`
- 以 `-` 开头的模式自动添加 `-e` 标志防止被解析为选项

```typescript
// tools/GrepTool/GrepTool.ts:378-384
// If pattern starts with dash, use -e flag to specify it as a pattern
if (pattern.startsWith('-')) {
  args.push('-e', pattern)
} else {
  args.push(pattern)
}
```

### path 参数

**可选参数**，指定搜索路径:

```typescript
// tools/GrepTool/GrepTool.ts:40-45
path: z.string().optional().describe(
  'File or directory to search in. Defaults to current working directory.'
)
```

**安全检查**:
- UNC 路径 (`\\` 或 `//` 开头) 跳过文件系统操作防止 NTLM 凭证泄露
- 不存在的路径返回错误并提供 cwd 建议

```typescript
// tools/GrepTool/GrepTool.ts:207-210
// SECURITY: Skip filesystem operations for UNC paths
if (absolutePath.startsWith('\\\\') || absolutePath.startsWith('//')) {
  return { result: true }
}
```

### glob 过滤

**可选参数**，文件名模式过滤:

```typescript
// tools/GrepTool/GrepTool.ts:47-51
glob: z.string().optional().describe(
  'Glob pattern to filter files (e.g. "*.js", "*.{ts,tsx}") - maps to rg --glob'
)
```

**解析逻辑**:
- 支持逗号和空格分隔多个模式
- 保留花括号模式不拆分: `*.{ts,tsx}` 保持完整

```typescript
// tools/GrepTool/GrepTool.ts:391-409
if (glob) {
  const globPatterns: string[] = []
  const rawPatterns = glob.split(/\s+/)
  
  for (const rawPattern of rawPatterns) {
    // If pattern contains braces, don't split further
    if (rawPattern.includes('{') && rawPattern.includes('}')) {
      globPatterns.push(rawPattern)
    } else {
      // Split on commas for patterns without braces
      globPatterns.push(...rawPattern.split(',').filter(Boolean))
    }
  }
  
  for (const globPattern of globPatterns.filter(Boolean)) {
    args.push('--glob', globPattern)
  }
}
```

### 输出模式

三种输出模式对应不同的使用场景:

| 模式 | 用途 | 输出格式 |
|------|------|----------|
| `files_with_matches` (默认) | 快速定位文件 | 文件路径列表 |
| `content` | 查看匹配内容 | 带行号的匹配行 |
| `count` | 统计匹配数量 | 文件:匹配数 |

```typescript
// tools/GrepTool/GrepTool.ts:52-57
output_mode: z.enum(['content', 'files_with_matches', 'count'])
  .optional().describe(
    'Output mode: "content" shows matching lines (supports -A/-B/-C context, -n line numbers, head_limit), "files_with_matches" shows file paths, "count" shows match counts'
  )
```

**模式对应的 ripgrep 标志**:

```typescript
// tools/GrepTool/GrepTool.ts:350-355
if (output_mode === 'files_with_matches') {
  args.push('-l')  // 只显示文件名
} else if (output_mode === 'count') {
  args.push('-c')  // 显示计数
}
```

### 上下文参数

仅 `content` 模式有效:

```typescript
// tools/GrepTool/GrepTool.ts:58-67
'-B': semanticNumber(z.number().optional()).describe(
  'Number of lines to show before each match'
),
'-A': semanticNumber(z.number().optional()).describe(
  'Number of lines to show after each match'
),
'-C': semanticNumber(z.number().optional()).describe('Alias for context'),
context: semanticNumber(z.number().optional()).describe(
  'Number of lines to show before and after each match'
)
```

**优先级**: `context` > `-C` > `-B`/`-A` 组合

```typescript
// tools/GrepTool/GrepTool.ts:362-376
if (output_mode === 'content') {
  if (context !== undefined) {
    args.push('-C', context.toString())
  } else if (context_c !== undefined) {
    args.push('-C', context_c.toString())
  } else {
    if (context_before !== undefined) {
      args.push('-B', context_before.toString())
    }
    if (context_after !== undefined) {
      args.push('-A', context_after.toString())
    }
  }
}
```

### 分页参数

**head_limit**: 结果数量限制

```typescript
// tools/GrepTool/GrepTool.ts:80-82
head_limit: semanticNumber(z.number().optional()).describe(
  'Limit output to first N lines/entries. Defaults to 250. Pass 0 for unlimited.'
)
```

**offset**: 跳过前 N 条结果

```typescript
// tools/GrepTool/GrepTool.ts:83-85
offset: semanticNumber(z.number().optional()).describe(
  'Skip first N lines/entries before applying head_limit'
)
```

**默认值与特殊处理**:

```typescript
// tools/GrepTool/GrepTool.ts:104-108
// Default cap on grep results when head_limit is unspecified.
// 250 is generous enough for exploratory searches while preventing context bloat.
// Pass head_limit=0 explicitly for unlimited.
const DEFAULT_HEAD_LIMIT = 250
```

```typescript
// tools/GrepTool/GrepTool.ts:114-128
function applyHeadLimit<T>(
  items: T[],
  limit: number | undefined,
  offset: number = 0
): { items: T[]; appliedLimit: number | undefined } {
  // Explicit 0 = unlimited escape hatch
  if (limit === 0) {
    return { items: items.slice(offset), appliedLimit: undefined }
  }
  const effectiveLimit = limit ?? DEFAULT_HEAD_LIMIT
  const sliced = items.slice(offset, offset + effectiveLimit)
  const wasTruncated = items.length - offset > effectiveLimit
  return {
    items: sliced,
    appliedLimit: wasTruncated ? effectiveLimit : undefined
  }
}
```

---

## 3. 搜索引擎实现

### ripgrep 集成

GrepTool 通过 `utils/ripgrep.ts` 调用 ripgrep:

```typescript
// tools/GrepTool/GrepTool.ts:21
import { ripGrep } from '../../utils/ripgrep.js'
```

**三种运行模式**:

```typescript
// utils/ripgrep.ts:24-29
type RipgrepConfig = {
  mode: 'system' | 'builtin' | 'embedded'
  command: string
  args: string[]
  argv0?: string
}
```

| 模式 | 说明 | 使用场景 |
|------|------|----------|
| `system` | 系统 ripgrep (`rg`) | `USE_BUILTIN_RIPGREP=false` 且系统已安装 |
| `builtin` | 内置 ripgrep | npm 安装时的平台特定二进制 |
| `embedded` | 嵌入式 ripgrep | Bun 打包模式 |

```typescript
// utils/ripgrep.ts:31-65
const getRipgrepConfig = memoize((): RipgrepConfig => {
  const userWantsSystemRipgrep = isEnvDefinedFalsy(
    process.env.USE_BUILTIN_RIPGREP
  )

  if (userWantsSystemRipgrep) {
    const { cmd: systemPath } = findExecutable('rg', [])
    if (systemPath !== 'rg') {
      // SECURITY: Use 'rg' instead of systemPath to prevent PATH hijacking
      return { mode: 'system', command: 'rg', args: [] }
    }
  }

  if (isInBundledMode()) {
    return {
      mode: 'embedded',
      command: process.execPath,
      args: ['--no-config'],
      argv0: 'rg'  // Bun 使用 argv0 区分嵌入式命令
    }
  }

  // builtin 模式使用平台特定二进制
  const rgRoot = path.resolve(__dirname, 'vendor', 'ripgrep')
  const command = process.platform === 'win32'
    ? path.resolve(rgRoot, `${process.arch}-win32`, 'rg.exe')
    : path.resolve(rgRoot, `${process.arch}-${process.platform}`, 'rg')

  return { mode: 'builtin', command, args: [] }
})
```

### 正则表达式支持

**ripgrep vs grep 语法差异**:

```typescript
// tools/GrepTool/prompt.ts:15
// Pattern syntax: Uses ripgrep (not grep) - literal braces need escaping
// use `interface\{\}` to find `interface{}` in Go code
```

**多行匹配**:

```typescript
// tools/GrepTool/GrepTool.ts:86-88
multiline: semanticBoolean(z.boolean().optional()).describe(
  'Enable multiline mode where . matches newlines. Default: false.'
)
```

```typescript
// tools/GrepTool/GrepTool.ts:340-343
if (multiline) {
  args.push('-U', '--multiline-dotall')
}
```

### 性能优化

**VCS 目录自动排除**:

```typescript
// tools/GrepTool/GrepTool.ts:93-102
// Version control system directories to exclude from searches
const VCS_DIRECTORIES_TO_EXCLUDE = [
  '.git', '.svn', '.hg', '.bzr', '.jj', '.sl'
] as const
```

```typescript
// tools/GrepTool/GrepTool.ts:330-335
const args = ['--hidden']  // 搜索隐藏文件但排除 VCS
for (const dir of VCS_DIRECTORIES_TO_EXCLUDE) {
  args.push('--glob', `!${dir}`)
}
```

**行长度限制**:

```typescript
// tools/GrepTool/GrepTool.ts:337-338
// Limit line length to prevent base64/minified content from cluttering output
args.push('--max-columns', '500')
```

**大缓冲区支持**:

```typescript
// utils/ripgrep.ts:80
const MAX_BUFFER_SIZE = 20_000_000 // 20MB; large monorepos can have 200k+ files
```

**EAGAIN 重试机制**:

```typescript
// utils/ripgrep.ts:393-409
// If we hit EAGAIN (resource temporarily unavailable), retry with single-threaded mode
if (!isRetry && isEagainError(stderr)) {
  logForDebugging(`rg EAGAIN error detected, retrying with -j 1`)
  ripGrepRaw(args, target, abortSignal, callback, true)  // 单线程重试
  return
}
```

---

## 4. 用户使用指南

### 如何高效搜索代码

**基本原则**:

1. **优先使用 GrepTool**: 永远不要用 Bash 的 `grep` 或 `rg`
2. **选择合适的输出模式**:
   - 不知道哪些文件 → `files_with_matches`
   - 查看具体代码 → `content`
   - 统计出现次数 → `count`

3. **使用 glob/type 缩小范围**:
   - `glob: "*.ts"` 比全项目搜索快得多
   - `type: "js"` 使用 ripgrep 内置类型定义

### 正则表达式技巧

**常用模式**:

```typescript
// 搜索函数定义
pattern: "function\\s+\\w+"

// 搜索日志错误
pattern: "log.*Error"

// 搜索 import 语句
pattern: "^import.*from"

// 搜索特定接口 (Go)
pattern: "interface\\{\\}"  // 注意转义花括号
```

**多行匹配示例**:

```typescript
// 搜索跨行的结构体字段
pattern: "struct \\{[\\s\\S]*?field"
multiline: true
```

### 大型项目搜索

**分页策略**:

```typescript
// 第一次搜索，获取前 250 条
{ pattern: "TODO", head_limit: 250 }

// 继续获取更多
{ pattern: "TODO", head_limit: 250, offset: 250 }

// 无限制搜索 (谨慎使用)
{ pattern: "specific-pattern", head_limit: 0 }
```

**路径限定**:

```typescript
// 只搜索特定目录
{ pattern: "function", path: "src/services" }

// 只搜索特定文件类型
{ pattern: "export", glob: "*.ts" }
```

### 最佳实践

**搜索流程建议**:

1. **先定位文件**: `output_mode: "files_with_matches"`
2. **再查看内容**: `output_mode: "content"` + 上下文
3. **最后阅读完整文件**: 使用 Read 工具

**避免常见错误**:

```typescript
// 错误: 直接用 bash grep
Bash: `grep -r "pattern" src/`

// 正确: 使用 GrepTool
Grep: { pattern: "pattern", path: "src" }
```

---

## 5. 结果处理

### 输出格式

**files_with_matches 模式**:

```typescript
// tools/GrepTool/GrepTool.ts:293-308
const result = `Found ${numFiles} ${plural(numFiles, 'file')}${limitInfo ? ` ${limitInfo}` : ''}\n${filenames.join('\n')}`
```

输出示例:
```
Found 3 files
src/services/user.ts
src/utils/format.ts
src/index.ts
```

**content 模式**:

```typescript
// tools/GrepTool/GrepTool.ts:267-278
const finalContent = limitInfo
  ? `${resultContent}\n\n[Showing results with pagination = ${limitInfo}]`
  : resultContent
```

输出示例:
```
src/services/user.ts:42:  function getUser() {
src/services/user.ts:45:    return userRepository.find(id)
src/utils/format.ts:12:  function formatDate(date: Date) {

[Showing results with pagination = limit: 250]
```

**count 模式**:

```typescript
// tools/GrepTool/GrepTool.ts:280-291
const summary = `\n\nFound ${matches} total ${matches === 1 ? 'occurrence' : 'occurrences'} across ${files} ${files === 1 ? 'file' : 'files'}.${limitInfo ? ` with pagination = ${limitInfo}` : ''}`
```

输出示例:
```
src/services/user.ts:15
src/utils/format.ts:8
src/index.ts:3

Found 26 total occurrences across 3 files.
```

### 行数限制

**默认值**:

```typescript
// tools/GrepTool/GrepTool.ts:104-108
const DEFAULT_HEAD_LIMIT = 250
```

**限制逻辑**:

```typescript
// tools/GrepTool/GrepTool.ts:110-128
function applyHeadLimit<T>(
  items: T[],
  limit: number | undefined,
  offset: number = 0
): { items: T[]; appliedLimit: number | undefined } {
  // head_limit=0 表示无限制
  if (limit === 0) {
    return { items: items.slice(offset), appliedLimit: undefined }
  }
  const effectiveLimit = limit ?? DEFAULT_HEAD_LIMIT
  const sliced = items.slice(offset, offset + effectiveLimit)
  // 只有实际截断时才报告 appliedLimit
  const wasTruncated = items.length - offset > effectiveLimit
  return {
    items: sliced,
    appliedLimit: wasTruncated ? effectiveLimit : undefined
  }
}
```

### 高亮显示

UI 组件处理结果显示:

```typescript
// tools/GrepTool/UI.tsx:16-118
function SearchResultSummary({
  count,
  countLabel,
  secondaryCount,
  secondaryLabel,
  content,
  verbose
}): React.ReactNode {
  // 非详细模式显示简洁摘要
  if (!verbose) {
    return (
      <MessageResponse height={1}>
        <Text>
          Found <Text bold>{count}</Text> {countLabel}
          {secondaryText}
          {count > 0 && <CtrlOToExpand />}
        </Text>
      </MessageResponse>
    )
  }
  // 详细模式显示完整内容
  return (
    <Box flexDirection="column">
      <Box flexDirection="row">
        <Text dimColor>&nbsp;&nbsp;⎿&nbsp;</Text>
        {primaryText}{secondaryText}
      </Box>
      <Box marginLeft={5}>
        <Text>{content}</Text>
      </Box>
    </Box>
  )
}
```

---

## 6. 与 GlobTool 配合

### 工具分工

| 工具 | 用途 | 搜索类型 |
|------|------|----------|
| **GlobTool** | 按文件名查找 | 文件路径模式 |
| **GrepTool** | 按内容搜索 | 文件内容正则 |

### 配合使用场景

**场景 1: 先找文件类型，再搜索内容**:

```typescript
// 1. 找出所有测试文件
Glob: { pattern: "**/*.test.ts" }

// 2. 在测试文件中搜索特定测试
Grep: { pattern: "describe.*UserService", glob: "*.test.ts" }
```

**场景 2: 搜索后确认文件**:

```typescript
// 1. 搜索包含特定代码的文件
Grep: { pattern: "interface User", output_mode: "files_with_matches" }

// 2. 查看文件完整内容
Read: { file_path: "src/types/user.ts" }
```

---

## 7. 从零实现

### 最小实现框架

```typescript
// 基础 GrepTool 实现
import { execFile } from 'child_process'
import { z } from 'zod'

const inputSchema = z.object({
  pattern: z.string(),
  path: z.string().optional(),
  glob: z.string().optional(),
  output_mode: z.enum(['content', 'files_with_matches', 'count']).optional()
})

async function grep(input: z.infer<typeof inputSchema>): Promise<string[]> {
  const args = ['--hidden']
  
  // 排除 VCS 目录
  args.push('--glob', '!.git')
  
  // 输出模式
  if (input.output_mode === 'files_with_matches') {
    args.push('-l')
  } else if (input.output_mode === 'count') {
    args.push('-c')
  }
  
  // 添加模式
  args.push(input.pattern)
  
  // glob 过滤
  if (input.glob) {
    args.push('--glob', input.glob)
  }
  
  // 目标路径
  const target = input.path || process.cwd()
  
  // 执行 ripgrep
  return new Promise((resolve, reject) => {
    execFile('rg', [...args, target], (error, stdout) => {
      if (error && error.code !== 1) {
        reject(error)
      } else {
        resolve(stdout.trim().split('\n').filter(Boolean))
      }
    })
  })
}
```

### 完整实现要点

**1. Schema 定义**:

```typescript
// tools/GrepTool/GrepTool.ts:33-89
const inputSchema = lazySchema(() =>
  z.strictObject({
    pattern: z.string(),
    path: z.string().optional(),
    glob: z.string().optional(),
    output_mode: z.enum(['content', 'files_with_matches', 'count']).optional(),
    '-B': z.number().optional(),
    '-A': z.number().optional(),
    '-C': z.number().optional(),
    context: z.number().optional(),
    '-n': z.boolean().optional(),
    '-i': z.boolean().optional(),
    type: z.string().optional(),
    head_limit: z.number().optional(),
    offset: z.number().optional(),
    multiline: z.boolean().optional()
  })
)
```

**2. 权限检查**:

```typescript
// tools/GrepTool/GrepTool.ts:233-240
async checkPermissions(input, context): Promise<PermissionDecision> {
  const appState = context.getAppState()
  return checkReadPermissionForTool(
    GrepTool,
    input,
    appState.toolPermissionContext
  )
}
```

**3. 路径验证**:

```typescript
// tools/GrepTool/GrepTool.ts:201-232
async validateInput({ path }): Promise<ValidationResult> {
  if (path) {
    const fs = getFsImplementation()
    const absolutePath = expandPath(path)
    
    // 安全检查: UNC 路径
    if (absolutePath.startsWith('\\\\')) {
      return { result: true }
    }
    
    try {
      await fs.stat(absolutePath)
    } catch (e: unknown) {
      if (isENOENT(e)) {
        return {
          result: false,
          message: `Path does not exist: ${path}`,
          errorCode: 1
        }
      }
      throw e
    }
  }
  return { result: true }
}
```

**4. ripgrep 调用**:

```typescript
// tools/GrepTool/GrepTool.ts:310-576
async call(input, context) {
  const absolutePath = input.path ? expandPath(input.path) : getCwd()
  const args = ['--hidden']
  
  // 排除 VCS
  for (const dir of VCS_DIRECTORIES_TO_EXCLUDE) {
    args.push('--glob', `!${dir}`)
  }
  
  // 行长度限制
  args.push('--max-columns', '500')
  
  // 多行模式
  if (input.multiline) {
    args.push('-U', '--multiline-dotall')
  }
  
  // 大小写
  if (input['-i']) {
    args.push('-i')
  }
  
  // 输出模式
  if (input.output_mode === 'files_with_matches') {
    args.push('-l')
  } else if (input.output_mode === 'count') {
    args.push('-c')
  }
  
  // 行号
  if (input['-n'] && input.output_mode === 'content') {
    args.push('-n')
  }
  
  // 模式处理
  if (input.pattern.startsWith('-')) {
    args.push('-e', input.pattern)
  } else {
    args.push(input.pattern)
  }
  
  // 文件类型
  if (input.type) {
    args.push('--type', input.type)
  }
  
  // glob
  if (input.glob) {
    args.push('--glob', input.glob)
  }
  
  // 执行搜索
  const results = await ripGrep(args, absolutePath, abortController.signal)
  
  // 处理结果...
}
```

**5. 结果排序** (files_with_matches 模式):

```typescript
// tools/GrepTool/GrepTool.ts:528-553
// 按修改时间排序
const stats = await Promise.allSettled(
  results.map(_ => getFsImplementation().stat(_))
)
const sortedMatches = results
  .map((_, i) => {
    const r = stats[i]!
    return [
      _,
      r.status === 'fulfilled' ? (r.value.mtimeMs ?? 0) : 0
    ] as const
  })
  .sort((a, b) => {
    // 测试环境按文件名排序保证确定性
    if (process.env.NODE_ENV === 'test') {
      return a[0].localeCompare(b[0])
    }
    // 按修改时间降序
    const timeComparison = b[1] - a[1]
    if (timeComparison === 0) {
      return a[0].localeCompare(b[0])
    }
    return timeComparison
  })
  .map(_ => _[0])
```

### ripgrep 工具函数

```typescript
// utils/ripgrep.ts:345-463
export async function ripGrep(
  args: string[],
  target: string,
  abortSignal: AbortSignal
): Promise<string[]> {
  return new Promise((resolve, reject) => {
    const handleResult = (error, stdout, stderr, isRetry) => {
      // 成功
      if (!error) {
        resolve(stdout.trim().split('\n').filter(Boolean))
        return
      }
      
      // 退出码 1 = 无匹配 (正常情况)
      if (error.code === 1) {
        resolve([])
        return
      }
      
      // EAGAIN 重试
      if (!isRetry && isEagainError(stderr)) {
        ripGrepRaw(args, target, abortSignal, callback, true)
        return
      }
      
      // 超时处理
      if (isTimeout && lines.length === 0) {
        reject(new RipgrepTimeoutError(
          `Ripgrep search timed out after 20 seconds`,
          lines
        ))
        return
      }
      
      resolve(lines)
    }
    
    ripGrepRaw(args, target, abortSignal, (error, stdout, stderr) => {
      handleResult(error, stdout, stderr, false)
    })
  })
}
```

---

## 总结

GrepTool 是 Claude Code CLI 中核心的代码搜索工具，其设计亮点包括:

1. **ripgrep 集成**: 高性能搜索引擎，支持系统/builtin/嵌入式三种模式
2. **安全设计**: UNC 路径跳过、PATH 劫持防护、权限检查
3. **性能优化**: VCS 目录排除、行长度限制、分页控制、EAGAIN 重试
4. **灵活输出**: 三种输出模式适配不同搜索场景
5. **用户体验**: React UI 组件、错误提示、路径建议

关键文件路径:
- `tools/GrepTool/GrepTool.ts` - 主工具实现
- `tools/GrepTool/prompt.ts` - 工具描述和使用指南
- `tools/GrepTool/UI.tsx` - UI 渲染组件
- `utils/ripgrep.ts` - ripgrep 封装和配置