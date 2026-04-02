# GlobTool 工具详解

**分析日期:** 2026-04-02

> **前置知识:** 阅读本文前，建议先阅读:
> - [03-工具系统设计](./03-工具系统设计.md) - 理解Tool接口与buildTool工厂

---

## 1. 概述

### GlobTool 的作用

GlobTool 是 Claude Code CLI 中用于**快速文件模式匹配**的核心工具。它基于 ripgrep (rg) 实现高性能文件搜索，能够在任意大小的代码库中快速定位文件。

**核心特性:**
- 支持标准 glob 模式语法（如 `**/*.js`、`src/**/*.ts`）
- 结果按文件修改时间排序（最旧的在前）
- 内置权限检查，遵循用户配置的读取限制
- 自动处理 `.gitignore` 和隐藏文件

**源码位置:**
- 主工具定义: `tools/GlobTool/GlobTool.ts`
- 核心算法: `utils/glob.ts`
- ripgrep 封装: `utils/ripgrep.ts`

### 与 GrepTool 的关系

GlobTool 和 GrepTool 是互补的搜索工具：
- **GlobTool**: 按**文件名模式**查找文件
- **GrepTool**: 按**文件内容**搜索匹配

当需要进行开放式的多轮搜索时，应考虑使用 Agent 工具而非直接调用 GlobTool。

---

## 2. 输入 Schema

### 参数定义

```typescript
// 源码位置: tools/GlobTool/GlobTool.ts:26-36
const inputSchema = z.strictObject({
  pattern: z.string().describe('The glob pattern to match files against'),
  path: z.string().optional().describe(
    'The directory to search in. If not specified, the current working directory will be used. ' +
    'IMPORTANT: Omit this field to use the default directory. ' +
    'DO NOT enter "undefined" or "null" - simply omit it for the default behavior. ' +
    'Must be a valid directory path if provided.'
  ),
})
```

### pattern 参数

**类型:** `string` (必需)

**作用:** 指定用于匹配文件名的 glob 模式。

**语法规则:**
| 模式 | 含义 | 示例 |
|------|------|------|
| `*` | 匹配任意字符（不含路径分隔符） | `*.js` 匹配所有 JS 文件 |
| `**` | 匹配零或多层目录 | `**/*.ts` 匹配所有子目录中的 TS 文件 |
| `?` | 匹配单个字符 | `file?.txt` 匹配 file1.txt、fileA.txt |
| `[abc]` | 匹配指定字符集中任意一个 | `file[123].txt` |
| `{a,b}` | 匹配其中任一模式 | `*.{js,ts}` 匹配 JS 或 TS 文件 |

**使用示例:**
```
**/*.js          # 所有 JS 文件
src/**/*.ts     # src 目录下所有 TS 文件
*.test.ts       # 所有测试文件
**/test/**      # 任意 test 目录下的所有文件
```

### path 参数

**类型:** `string` (可选)

**作用:** 指定搜索的起始目录。

**行为:**
- 省略时使用当前工作目录 (`getCwd()`)
- 支持相对路径和绝对路径
- 支持用户主目录展开 (`~` -> `homedir()`)
- 必须是有效的目录路径（文件路径会导致验证失败）

**路径验证逻辑** (源码位置: `GlobTool.ts:94-134`):
```typescript
async validateInput({ path }): Promise<ValidationResult> {
  if (path) {
    const absolutePath = expandPath(path)
    
    // SECURITY: 跳过 UNC 路径的文件系统操作，防止 NTLM 凭据泄露
    if (absolutePath.startsWith('\\\\') || absolutePath.startsWith('//')) {
      return { result: true }
    }
    
    // 检查路径是否存在
    const stats = await fs.stat(absolutePath)
    
    // 检查是否为目录
    if (!stats.isDirectory()) {
      return { result: false, message: 'Path is not a directory: ...' }
    }
  }
  return { result: true }
}
```

---

## 3. 匹配算法

### 核心实现原理

GlobTool 不直接实现 glob 匹配算法，而是将任务委托给 **ripgrep** 的 `--files --glob` 功能。这种设计利用了 ripgrep 高度优化的文件遍历引擎。

**调用链:**
```
GlobTool.call() → glob() → ripGrep() → rg --files --glob <pattern>
```

### glob 函数解析

**源码位置:** `utils/glob.ts:66-130`

```typescript
export async function glob(
  filePattern: string,
  cwd: string,
  { limit, offset }: { limit: number; offset: number },
  abortSignal: AbortSignal,
  toolPermissionContext: ToolPermissionContext,
): Promise<{ files: string[]; truncated: boolean }>
```

**处理流程:**

1. **绝对路径处理** (第 78-84 行):
   ```typescript
   if (isAbsolute(filePattern)) {
     const { baseDir, relativePattern } = extractGlobBaseDirectory(filePattern)
     if (baseDir) {
       searchDir = baseDir
       searchPattern = relativePattern
     }
   }
   ```
   ripgrep 的 `--glob` 只支持相对模式，因此需要提取基准目录。

2. **构建 ripgrep 参数** (第 98-117 行):
   ```typescript
   const args = [
     '--files',           // 列出文件而非搜索内容
     '--glob', searchPattern,  // 应用 glob 模式
     '--sort=modified',   // 按修改时间排序
     ...(noIgnore ? ['--no-ignore'] : []),  // 忽略 .gitignore
     ...(hidden ? ['--hidden'] : []),        // 包含隐藏文件
   ]
   ```

3. **添加排除模式**:
   ```typescript
   // 添加权限拒绝规则
   for (const pattern of ignorePatterns) {
     args.push('--glob', `!${pattern}`)
   }
   // 排除孤立插件缓存
   for (const exclusion of await getGlobExclusionsForPluginCache(searchDir)) {
     args.push('--glob', exclusion)
   }
   ```

### extractGlobBaseDirectory 函数

**源码位置:** `utils/glob.ts:17-64`

**作用:** 从 glob 模式中提取静态基准目录。

```typescript
export function extractGlobBaseDirectory(pattern: string): {
  baseDir: string
  relativePattern: string
}
```

**算法逻辑:**
1. 查找第一个 glob 特殊字符 (`*`, `?`, `[`, `{`) 的位置
2. 提取该位置之前的静态前缀
3. 在静态前缀中找到最后一个路径分隔符
4. 分割为 `baseDir` 和 `relativePattern`

**示例:**
| 输入模式 | baseDir | relativePattern |
|----------|---------|-----------------|
| `/home/user/src/**/*.ts` | `/home/user/src` | `**/*.ts` |
| `*.js` | `` (空) | `*.js` |
| `src/**/*.test.ts` | `src` | `**/*.test.ts` |

### 性能优化

**1. 结果限制**
```typescript
const limit = globLimits?.maxResults ?? 100  // 默认限制 100 个结果
const truncated = absolutePaths.length > offset + limit
const files = absolutePaths.slice(offset, offset + limit)
```

**2. 路径相对化**
```typescript
const filenames = files.map(toRelativePath)  // 相对路径节省 token
```

**3. ripgrep 多线程优化**

ripgrep 自动并行处理，在大型仓库中性能优异。当遇到资源限制错误 (EAGAIN) 时，会自动降级为单线程模式重试：

```typescript
// 源码位置: utils/ripgrep.ts:394-409
if (!isRetry && isEagainError(stderr)) {
  ripGrepRaw(args, target, abortSignal, callback, true)  // single-threaded
}
```

### 排除模式

**环境变量控制:**
| 变量 | 默认值 | 作用 |
|------|--------|------|
| `CLAUDE_CODE_GLOB_NO_IGNORE` | `true` | 是否忽略 `.gitignore` |
| `CLAUDE_CODE_GLOB_HIDDEN` | `true` | 是否包含隐藏文件 |

**权限排除:**

GlobTool 会自动排除用户拒绝读取的文件：
```typescript
// 源码位置: utils/glob.ts:86-89
const ignorePatterns = normalizePatternsToPath(
  getFileReadIgnorePatterns(toolPermissionContext),
  searchDir,
)
```

---

## 4. 用户使用指南

### 快速查找文件

**基本用法:**
```json
{
  "pattern": "**/*.ts"
}
```

**指定搜索目录:**
```json
{
  "pattern": "*.test.ts",
  "path": "src"
}
```

### 模式语法详解

**通配符:**
```
*       匹配任意数量字符（不含 /）
**      匹配零或多个目录层级
?       匹配单个字符
[abc]   匹配 a、b 或 c
[a-z]   匹配 a 到 z 范围内的字符
[!abc]  匹配非 a、b、c 的字符
```

**大括号扩展:**
```
*.{js,ts}       匹配所有 .js 或 .ts 文件
src/{lib,util}/*.ts  匹配 src/lib/*.ts 和 src/util/*.ts
```

**否定模式 (ripgrep 特定):**
```
!node_modules   排除 node_modules 目录
!*.min.js       排除压缩文件
```

### 常见模式示例

| 需求 | 模式 |
|------|------|
| 查找所有 TypeScript 文件 | `**/*.ts` |
| 查找所有测试文件 | `**/*.test.ts` 或 `**/*.spec.ts` |
| 查找特定目录下的文件 | `src/components/**/*.tsx` |
| 查找配置文件 | `**/config*.json` 或 `**/*.config.js` |
| 查找 React 组件 | `**/*.{tsx,jsx}` |
| 排除 node_modules | 默认已排除 (可通过环境变量控制) |

### 最佳实践

**1. 优先使用具体路径**
```
# 好 - 搜索特定目录
pattern: "src/**/*.ts"

# 避免 - 搜索整个文件系统
pattern: "**/*.ts"  # 可能返回大量结果并被截断
```

**2. 结合 GrepTool 使用**
```
# 1. 先用 GlobTool 找到目标文件
pattern: "**/api/**/*.ts"

# 2. 再用 GrepTool 搜索内容
pattern: "export.*fetch"
glob: "api/**/*.ts"
```

**3. 利用结果排序**
结果按修改时间排序，最近修改的文件在最后。如果需要最新文件，可反向遍历结果。

**4. 处理截断结果**
当返回 `(Results are truncated...)` 时：
- 使用更具体的 `path` 参数
- 使用更精确的 `pattern`
- 分批查询

---

## 5. 与 GrepTool 配合

### 工具协作模式

GlobTool 和 GrepTool 是 Claude Code 的两大搜索支柱：

| 工具 | 搜索维度 | 适用场景 |
|------|----------|----------|
| GlobTool | 文件名 | "找到所有测试文件" |
| GrepTool | 文件内容 | "找到包含 'useEffect' 的代码" |

### 组合使用策略

**场景 1: 定位特定文件类型中的特定内容**
```json
// 步骤 1: GlobTool 确定范围
{ "pattern": "**/hooks/**/*.ts" }

// 步骤 2: GrepTool 搜索内容
{ "pattern": "useState", "glob": "**/hooks/*.ts" }
```

**场景 2: 找到文件后读取内容**
```json
// 步骤 1: GlobTool 找到文件
{ "pattern": "**/package.json" }

// 步骤 2: ReadTool 读取文件
{ "file_path": "/path/to/package.json" }
```

### UI 共享组件

GlobTool 的 UI 组件复用了 GrepTool 的结果渲染：

```typescript
// 源码位置: tools/GlobTool/UI.tsx:53
export const renderToolResultMessage = GrepTool.renderToolResultMessage
```

这确保了两种搜索工具在终端中的一致显示体验。

---

## 6. 大型项目优化

### 结果限制机制

**默认限制:** 100 个文件

```typescript
// 源码位置: tools/GlobTool/GlobTool.ts:157
const limit = globLimits?.maxResults ?? 100
```

**截断提示:**
当结果被截断时，返回会包含提示：
```
(Results are truncated. Consider using a more specific path or pattern.)
```

### ripgrep 超时配置

```typescript
// 源码位置: utils/ripgrep.ts:130-133
const defaultTimeout = getPlatform() === 'wsl' ? 60_000 : 20_000
const timeout = parsedSeconds > 0 ? parsedSeconds * 1000 : defaultTimeout
```

可通过环境变量自定义超时时间：
```bash
CLAUDE_CODE_GLOB_TIMEOUT_SECONDS=30  # 30 秒超时
```

### 内存优化

ripgrep 使用流式处理，避免将所有结果加载到内存：

```typescript
// 源码位置: utils/ripgrep.ts:80
const MAX_BUFFER_SIZE = 20_000_000  // 20MB 缓冲区上限
```

对于超大仓库（如 247k 文件），ripgrep 不会一次性加载所有路径，而是流式返回结果。

### 单线程降级

当系统资源受限（如 Docker 容器、CI 环境）导致 EAGAIN 错误时：

```typescript
// 源码位置: utils/ripgrep.ts:394-409
if (!isRetry && isEagainError(stderr)) {
  logEvent('tengu_ripgrep_eagain_retry', {})
  ripGrepRaw(args, target, abortSignal, callback, true)  // -j 1 单线程
}
```

---

## 7. 从零实现

### 最小实现

以下是 GlobTool 的核心逻辑简化版：

```typescript
import { glob as rgGlob } from 'ripgrep'
import { join, isAbsolute, dirname, basename } from 'path'

interface GlobResult {
  files: string[]
  truncated: boolean
}

async function glob(
  pattern: string,
  cwd: string,
  limit: number = 100
): Promise<GlobResult> {
  let searchDir = cwd
  let searchPattern = pattern

  // 处理绝对路径
  if (isAbsolute(pattern)) {
    const { baseDir, relativePattern } = extractBaseDir(pattern)
    if (baseDir) {
      searchDir = baseDir
      searchPattern = relativePattern
    }
  }

  // 调用 ripgrep
  const args = [
    '--files',
    '--glob', searchPattern,
    '--sort=modified',
    '--no-ignore',
    '--hidden'
  ]

  const results = await execRipgrep(args, searchDir)
  const truncated = results.length > limit
  const files = results.slice(0, limit)

  return { files, truncated }
}

function extractBaseDir(pattern: string): { baseDir: string; relativePattern: string } {
  const globChars = /[*?[{]/
  const match = pattern.match(globChars)

  if (!match || match.index === undefined) {
    return { baseDir: dirname(pattern), relativePattern: basename(pattern) }
  }

  const staticPrefix = pattern.slice(0, match.index)
  const lastSep = Math.max(staticPrefix.lastIndexOf('/'), staticPrefix.lastIndexOf('\\'))

  if (lastSep === -1) {
    return { baseDir: '', relativePattern: pattern }
  }

  return {
    baseDir: staticPrefix.slice(0, lastSep),
    relativePattern: pattern.slice(lastSep + 1)
  }
}
```

### 工具注册

```typescript
// tools/GlobTool/GlobTool.ts 完整结构
export const GlobTool = buildTool({
  name: 'Glob',
  
  // Schema 定义
  inputSchema: z.strictObject({
    pattern: z.string(),
    path: z.string().optional()
  }),
  
  outputSchema: z.object({
    durationMs: z.number(),
    numFiles: z.number(),
    filenames: z.array(z.string()),
    truncated: z.boolean()
  }),
  
  // 工具特性
  isConcurrencySafe: () => true,   // 可并发调用
  isReadOnly: () => true,          // 只读操作
  
  // 权限检查
  async checkPermissions(input, context) {
    return checkReadPermissionForTool(GlobTool, input, context)
  },
  
  // 路径解析
  getPath({ path }) {
    return path ? expandPath(path) : getCwd()
  },
  
  // 核心执行逻辑
  async call(input, { abortController, getAppState, globLimits }) {
    const start = Date.now()
    const limit = globLimits?.maxResults ?? 100
    
    const { files, truncated } = await glob(
      input.pattern,
      GlobTool.getPath(input),
      { limit, offset: 0 },
      abortController.signal,
      getAppState().toolPermissionContext
    )
    
    return {
      data: {
        filenames: files.map(toRelativePath),
        durationMs: Date.now() - start,
        numFiles: files.length,
        truncated
      }
    }
  },
  
  // 结果格式化
  mapToolResultToToolResultBlockParam(output, toolUseID) {
    if (output.filenames.length === 0) {
      return { tool_use_id: toolUseID, type: 'tool_result', content: 'No files found' }
    }
    
    const content = [
      ...output.filenames,
      ...(output.truncated ? ['(Results are truncated...)'] : [])
    ].join('\n')
    
    return { tool_use_id: toolUseID, type: 'tool_result', content }
  }
})
```

### 关键依赖

```
utils/glob.ts           # glob 函数实现
utils/ripgrep.ts        # ripgrep 进程封装
utils/path.ts           # 路径处理工具
utils/permissions/      # 权限检查系统
Tool.ts                 # buildTool 工厂函数
```

---

## 附录: 输出格式

### 成功响应

```json
{
  "durationMs": 45,
  "numFiles": 23,
  "filenames": [
    "src/index.ts",
    "src/utils/glob.ts",
    "src/utils/path.ts"
  ],
  "truncated": false
}
```

### 无匹配结果

```json
{
  "tool_use_id": "toolu_xxx",
  "type": "tool_result",
  "content": "No files found"
}
```

### 结果截断

```
src/file1.ts
src/file2.ts
...
src/file100.ts
(Results are truncated. Consider using a more specific path or pattern.)
```

---

*文档版本: 1.0.0 | 基于 Claude Code CLI 源码分析*