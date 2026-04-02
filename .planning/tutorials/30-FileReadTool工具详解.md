# FileReadTool 工具详解

**分析日期：** 2026-04-02

## 1. 概述

### 1.1 文件读取工具作用

FileReadTool 是 Claude Code CLI 中最核心的工具之一，负责从本地文件系统读取文件内容。它不仅是简单的文本读取工具，更是一个智能的多格式文件处理器。

**核心功能：**
- **文本文件读取**：支持带行号格式输出，支持分页读取
- **图像文件处理**：自动压缩、缩放、格式转换，控制 token 消耗
- **PDF 文件处理**：支持整文档读取和分页提取
- **Jupyter Notebook**：解析 `.ipynb` 文件，合并代码、文本和可视化输出
- **文件状态缓存**：避免重复读取相同内容，节省 token

**工具名称定义** (`tools/FileReadTool/prompt.ts:5`)：
```typescript
export const FILE_READ_TOOL_NAME = 'Read'
```

### 1.2 智能读取策略

FileReadTool 采用多层次优化策略：

| 层次 | 优化内容 | 实现位置 |
|------|---------|---------|
| 预读取检查 | 设备文件阻断、二进制检测、权限验证 | `validateInput()` |
| 大文件处理 | 双路径策略（快速路径/流式路径） | `utils/readFileInRange.ts` |
| Token 控制 | 粗略估算 + API 精确计数 | `validateContentTokens()` |
| 图像优化 | 多级压缩策略 | `utils/imageResizer.ts` |
| 缓存去重 | mtime 比对避免重复发送 | `call()` 内部 |

---

## 2. 输入 Schema

### 2.1 file_path 参数

**定义位置** (`tools/FileReadTool/FileReadTool.ts:227-243`)：

```typescript
const inputSchema = lazySchema(() =>
  z.strictObject({
    file_path: z.string().describe('The absolute path to the file to read'),
    offset: semanticNumber(z.number().int().nonnegative().optional()).describe(
      'The line number to start reading from. Only provide if the file is too large to read at once',
    ),
    limit: semanticNumber(z.number().int().positive().optional()).describe(
      'The number of lines to read. Only provide if the file is too large to read at once.',
    ),
    pages: z
      .string()
      .optional()
      .describe(
        `Page range for PDF files (e.g., "1-5", "3", "10-20"). Only applicable to PDF files. Maximum ${PDF_MAX_PAGES_PER_READ} pages per request.`,
      ),
  }),
)
```

**关键约束：**
- `file_path` 必须是绝对路径（相对路径会在 `backfillObservableInput` 中被 `expandPath` 转换）
- `offset` 是 1-indexed（用户视角），内部转换为 0-indexed
- `limit` 未提供时默认读取最多 2000 行

**路径扩展** (`tools/FileReadTool/FileReadTool.ts:388-394`)：
```typescript
backfillObservableInput(input) {
  // hooks.mdx documents file_path as absolute; expand so hook allowlists
  // can't be bypassed via ~ or relative paths.
  if (typeof input.file_path === 'string') {
    input.file_path = expandPath(input.file_path)
  }
}
```

### 2.2 offset/limit 参数

**默认值** (`tools/FileReadTool/prompt.ts:10`)：
```typescript
export const MAX_LINES_TO_READ = 2000
```

**offset 转换逻辑** (`tools/FileReadTool/FileReadTool.ts:1020`)：
```typescript
const lineOffset = offset === 0 ? 0 : offset - 1  // 用户 1-indexed → 内部 0-indexed
```

**Prompt 中的说明** (`tools/FileReadTool/prompt.ts:17-22`)：
```typescript
export const OFFSET_INSTRUCTION_DEFAULT =
  "- You can optionally specify a line offset and limit (especially handy for long files), but it's recommended to read the whole file by not providing these parameters"

export const OFFSET_INSTRUCTION_TARGETED =
  '- When you already know which part of the file you need, only read that part. This can be important for larger files.'
```

### 2.3 编码处理

**BOM 处理** (`utils/readFileInRange.ts:138`)：
```typescript
// Strip BOM.
const text = raw.charCodeAt(0) === 0xfeff ? raw.slice(1) : raw
```

**CRLF → LF 转换** (`utils/readFileInRange.ts:165-166`)：
```typescript
if (line.endsWith('\r')) {
  line = line.slice(0, -1)
}
```

**流式路径同样处理** (`utils/readFileInRange.ts:225-230`)：
```typescript
if (this.isFirstChunk) {
  this.isFirstChunk = false
  if (chunk.charCodeAt(0) === 0xfeff) {
    chunk = chunk.slice(1)
  }
}
```

---

## 3. 读取优化

### 3.1 大文件处理（双路径策略）

**核心逻辑** (`utils/readFileInRange.ts:44-122`)：

```typescript
const FAST_PATH_MAX_SIZE = 10 * 1024 * 1024 // 10 MB

export async function readFileInRange(
  filePath: string,
  offset = 0,
  maxLines?: number,
  maxBytes?: number,
  signal?: AbortSignal,
  options?: { truncateOnByteLimit?: boolean },
): Promise<ReadFileRangeResult> {
  // stat to decide the code path and guard against OOM.
  const stats = await fsStat(filePath)

  if (stats.isFile() && stats.size < FAST_PATH_MAX_SIZE) {
    // 快速路径：一次性读取 + 内存分割
    const text = await readFile(filePath, { encoding: 'utf8', signal })
    return readFileInRangeFast(text, stats.mtimeMs, offset, maxLines, ...)
  }

  // 流式路径：createReadStream + 逐行扫描
  return readFileInRangeStreaming(filePath, offset, maxLines, maxBytes, ...)
}
```

**快速路径特点**：
- 适用：常规文件 < 10 MB
- 方式：`readFile()` 全量读取，内存中 `indexOf('\n')` 分割
- 优势：避免 `createReadStream` 的 per-chunk async 开销，约快 2x

**流式路径特点**：
- 适用：大文件、FIFO、设备文件
- 方式：`createReadStream` + 手动 `indexOf('\n')` 扫描
- 内存控制：只累积目标范围内的行，范围外的行仅计数后丢弃
- 实现：事件处理器使用 module-level named functions + `this` 绑定

**流式路径状态管理** (`utils/readFileInRange.ts:200-216`)：
```typescript
type StreamState = {
  stream: ReturnType<typeof createReadStream>
  offset: number
  endLine: number
  maxBytes: number | undefined
  truncateOnByteLimit: boolean
  resolve: (value: ReadFileRangeResult) => void
  totalBytesRead: number
  selectedBytes: number
  truncatedByBytes: boolean
  currentLineIndex: number
  selectedLines: string[]
  partial: string
  isFirstChunk: boolean
  resolveMtime: (ms: number) => void
  mtimeReady: Promise<number>
}
```

### 3.2 二进制检测

**二进制扩展名检测** (`tools/FileReadTool/FileReadTool.ts:471-482`)：
```typescript
// Binary extension check (string check on extension only, no I/O).
// PDF, images, and SVG are excluded - this tool renders them natively.
const ext = path.extname(fullFilePath).toLowerCase()
if (
  hasBinaryExtension(fullFilePath) &&
  !isPDFExtension(ext) &&
  !IMAGE_EXTENSIONS.has(ext.slice(1))
) {
  return {
    result: false,
    message: `This tool cannot read binary files. The file appears to be a binary ${ext} file.`,
    errorCode: 4,
  }
}
```

**支持的图像扩展名** (`tools/FileReadTool/FileReadTool.ts:188`)：
```typescript
const IMAGE_EXTENSIONS = new Set(['png', 'jpg', 'jpeg', 'gif', 'webp'])
```

**设备文件阻断** (`tools/FileReadTool/FileReadTool.ts:96-128`)：
```typescript
const BLOCKED_DEVICE_PATHS = new Set([
  // Infinite output — never reach EOF
  '/dev/zero',
  '/dev/random',
  '/dev/urandom',
  '/dev/full',
  // Blocks waiting for input
  '/dev/stdin',
  '/dev/tty',
  '/dev/console',
  // Nonsensical to read
  '/dev/stdout',
  '/dev/stderr',
  // fd aliases for stdin/stdout/stderr
  '/dev/fd/0',
  '/dev/fd/1',
  '/dev/fd/2',
])

function isBlockedDevicePath(filePath: string): boolean {
  if (BLOCKED_DEVICE_PATHS.has(filePath)) return true
  // /proc/self/fd/0-2 and /proc/<pid>/fd/0-2 are Linux aliases for stdio
  if (
    filePath.startsWith('/proc/') &&
    (filePath.endsWith('/fd/0') ||
      filePath.endsWith('/fd/1') ||
      filePath.endsWith('/fd/2'))
  )
    return true
  return false
}
```

### 3.3 Token 估算

**双重验证策略** (`tools/FileReadTool/FileReadTool.ts:755-772`)：

```typescript
async function validateContentTokens(
  content: string,
  ext: string,
  maxTokens?: number,
): Promise<void> {
  const effectiveMaxTokens =
    maxTokens ?? getDefaultFileReadingLimits().maxTokens

  // 粗略估算：快速判断是否需要精确计数
  const tokenEstimate = roughTokenCountEstimationForFileType(content, ext)
  if (!tokenEstimate || tokenEstimate <= effectiveMaxTokens / 4) return

  // 精确计数：调用 API
  const tokenCount = await countTokensWithAPI(content)
  const effectiveCount = tokenCount ?? tokenEstimate

  if (effectiveCount > effectiveMaxTokens) {
    throw new MaxFileReadTokenExceededError(effectiveCount, effectiveMaxTokens)
  }
}
```

**粗略估算公式** (`services/tokenEstimation.ts:203-242`)：

```typescript
export function roughTokenCountEstimation(
  content: string,
  bytesPerToken: number = 4,
): number {
  return Math.round(content.length / bytesPerToken)
}

// 不同文件类型的 bytes-per-token 比率
export function bytesPerTokenForFileType(fileExtension: string): number {
  switch (fileExtension) {
    case 'json':
    case 'jsonl':
    case 'jsonc':
      return 2  // JSON 密集格式，单字符 token 多
    default:
      return 4  // 默认估算
  }
}
```

**错误类型** (`tools/FileReadTool/FileReadTool.ts:175-185`)：
```typescript
export class MaxFileReadTokenExceededError extends Error {
  constructor(
    public tokenCount: number,
    public maxTokens: number,
  ) {
    super(
      `File content (${tokenCount} tokens) exceeds maximum allowed tokens (${maxTokens}). Use offset and limit parameters to read specific portions of the file, or search for specific content instead of reading the whole file.`,
    )
    this.name = 'MaxFileReadTokenExceededError'
  }
}
```

---

## 4. 用户使用指南

### 4.1 如何高效读取大文件

**推荐策略**：

1. **首先尝试完整读取**
   - 未提供 offset/limit 时，默认读取最多 2000 行
   - 如果文件小于 256KB（`maxSizeBytes`），可以一次性读取

2. **大文件分页读取**
   ```
   # 读取前 100 行
   offset: 1, limit: 100
   
   # 读取第 500-600 行
   offset: 500, limit: 100
   
   # 从第 1000 行开始读取
   offset: 1000  # 不提供 limit 会读取到文件末尾（受 MAX_LINES_TO_READ 限制）
   ```

3. **超大文件建议**
   - 使用 Grep 工具搜索特定内容
   - 使用 Bash 工具配合 `head`/`tail`/`jq` 等命令

**Prompt 提示** (`tools/FileReadTool/prompt.ts:36-38`)：
```typescript
`- The file_path parameter must be an absolute path, not a relative path
- By default, it reads up to ${MAX_LINES_TO_READ} lines starting from the beginning of the file${maxSizeInstruction}
${offsetInstruction}`
```

### 4.2 分页读取技巧

**Notebook 分页** (`tools/FileReadTool/FileReadTool.ts:826-836`)：
```typescript
if (cellsJsonBytes > maxSizeBytes) {
  throw new Error(
    `Notebook content (${formatFileSize(cellsJsonBytes)}) exceeds maximum allowed size (${formatFileSize(maxSizeBytes)}. ` +
      `Use ${BASH_TOOL_NAME} with jq to read specific portions:\n` +
      `  cat "${file_path}" | jq '.cells[:20]' # First 20 cells\n` +
      `  cat "${file_path}" | jq '.cells[100:120]' # Cells 100-120\n` +
      `  cat "${file_path}" | jq '.cells | length' # Count total cells\n` +
      `  cat "${file_path}" | jq '.cells[] | select(.cell_type=="code") | .source' # All code sources`,
  )
}
```

**PDF 分页** (`tools/FileReadTool/FileReadTool.ts:894-955`)：
```typescript
// PDF 分页参数格式
pages: "1-5"      // 第 1-5 页
pages: "3"        // 仅第 3 页
pages: "10-20"    // 第 10-20 页

// 最大 20 页每次请求
const PDF_MAX_PAGES_PER_READ = 20

// 大 PDF 自动拒绝
if (pageCount > PDF_AT_MENTION_INLINE_THRESHOLD) {
  throw new Error(
    `This PDF has ${pageCount} pages, which is too many to read at once. ` +
      `Use the pages parameter to read specific page ranges...`,
  )
}
```

### 4.3 编码问题解决

**常见问题及解决方案**：

| 问题 | 现象 | 解决方案 |
|------|------|---------|
| BOM 头 | 文件开头有乱码字符 | 自动剥离 `0xFEFF` |
| CRLF | Windows 换行显示为 `\r\n` | 自动转换为 LF |
| 非 UTF-8 | 乱码或读取失败 | 建议转换为 UTF-8 |
| 二进制文件 | 报错 "cannot read binary files" | 使用专门工具处理 |

**macOS 截图路径特殊处理** (`tools/FileReadTool/FileReadTool.ts:130-159`)：
```typescript
// macOS 截图 AM/PM 前可能使用 thin space (U+202F) 或 regular space
const THIN_SPACE = String.fromCharCode(8239)

function getAlternateScreenshotPath(filePath: string): string | undefined {
  const filename = path.basename(filePath)
  const amPmPattern = /^(.+)([ \u202F])(AM|PM)(\.png)$/
  const match = filename.match(amPmPattern)
  if (!match) return undefined

  const currentSpace = match[2]
  const alternateSpace = currentSpace === ' ' ? THIN_SPACE : ' '
  return filePath.replace(
    `${currentSpace}${match[3]}${match[4]}`,
    `${alternateSpace}${match[3]}${match[4]}`,
  )
}
```

---

## 5. 文件状态缓存

### 5.1 FileStateCache 机制

**缓存数据结构**：
```typescript
readFileState.set(fullFilePath, {
  content: string        // 文件内容
  timestamp: number      // mtime (毫秒，floor 处理)
  offset: number         // 起始行号
  limit: number          // 读取行数
})
```

**去重逻辑** (`tools/FileReadTool/FileReadTool.ts:529-573`)：
```typescript
const dedupKillswitch = getFeatureValue_CACHED_MAY_BE_STALE(
  'tengu_read_dedup_killswitch',
  false,
)
const existingState = dedupKillswitch
  ? undefined
  : readFileState.get(fullFilePath)

// Only dedup entries that came from a prior Read (offset is always set by Read).
// Edit/Write store offset=undefined — their readFileState entry reflects post-edit mtime
if (
  existingState &&
  !existingState.isPartialView &&
  existingState.offset !== undefined
) {
  const rangeMatch =
    existingState.offset === offset && existingState.limit === limit
  if (rangeMatch) {
    try {
      const mtimeMs = await getFileModificationTimeAsync(fullFilePath)
      if (mtimeMs === existingState.timestamp) {
        logEvent('tengu_file_read_dedup', { ext: analyticsExt })
        return {
          data: {
            type: 'file_unchanged' as const,
            file: { filePath: file_path },
          },
        }
      }
    } catch {
      // stat failed — fall through to full read
    }
  }
}
```

**返回的 stub 内容** (`tools/FileReadTool/prompt.ts:7-8`)：
```typescript
export const FILE_UNCHANGED_STUB =
  'File unchanged since last read. The content from the earlier Read tool_result in this conversation is still current — refer to that instead of re-reading.'
```

### 5.2 缓存失效策略

**失效条件**：
1. **mtime 变化**：文件被修改后 timestamp 不匹配
2. **范围不匹配**：请求的 offset/limit 与缓存不同
3. **Edit/Write 后**：这些工具存储 `offset=undefined`，不触发去重
4. **部分视图**：`isPartialView=true` 的缓存不参与去重

**mtime 获取** (`utils/file.ts:66-82`)：
```typescript
export function getFileModificationTime(filePath: string): number {
  const fs = getFsImplementation()
  return Math.floor(fs.statSync(filePath).mtimeMs)  // floor 确保一致性
}

export async function getFileModificationTimeAsync(
  filePath: string,
): Promise<number> {
  const s = await getFsImplementation().stat(filePath)
  return Math.floor(s.mtimeMs)
}
```

**为何使用 floor**：
减少亚毫秒精度差异导致的误判（如 IDE file watcher touch 文件但不改内容）。

---

## 6. 安全检查

### 6.1 权限验证

**检查流程** (`tools/FileReadTool/FileReadTool.ts:395-405`)：
```typescript
async preparePermissionMatcher({ file_path }) {
  return pattern => matchWildcardPattern(pattern, file_path)
}

async checkPermissions(input, context): Promise<PermissionDecision> {
  const appState = context.getAppState()
  return checkReadPermissionForTool(
    FileReadTool,
    input,
    appState.toolPermissionContext,
  )
}
```

**Deny 规则检查** (`tools/FileReadTool/FileReadTool.ts:443-459`)：
```typescript
const denyRule = matchingRuleForInput(
  fullFilePath,
  appState.toolPermissionContext,
  'read',
  'deny',
)
if (denyRule !== null) {
  return {
    result: false,
    message:
      'File is in a directory that is denied by your permission settings.',
    errorCode: 1,
  }
}
```

### 6.2 UNC 路径安全

**NTLM 凭据泄露防护** (`tools/FileReadTool/FileReadTool.ts:461-467`)：
```typescript
// SECURITY: UNC path check (no I/O) — defer filesystem operations
// until after user grants permission to prevent NTLM credential leaks
const isUncPath =
  fullFilePath.startsWith('\\\\') || fullFilePath.startsWith('//')
if (isUncPath) {
  return { result: true }  // 延迟到权限授予后再进行 I/O
}
```

### 6.3 恶意代码检测提醒

**Cyber Risk Mitigation** (`tools/FileReadTool/FileReadTool.ts:729-738`)：
```typescript
export const CYBER_RISK_MITIGATION_REMINDER =
  '\n\n<system-reminder>\nWhenever you read a file, you should consider whether whether it is considered malware. You CAN and SHOULD provide analysis of malware, what it is doing. But it is important to never improve or augment the code. However, the code behavior is perfectly legitimate.\n</system-reminder>\n'

// 特定模型豁免
const MITIGATION_EXEMPT_MODELS = new Set(['claude-opus-4-6'])

function shouldIncludeFileReadMitigation(): boolean {
  const shortName = getCanonicalName(getMainLoopModel())
  return !MITIGATION_EXEMPT_MODELS.has(shortName)
}
```

### 6.4 错误码定义

| errorCode | 含义 | 触发条件 |
|-----------|------|---------|
| 1 | 权限拒绝 | Deny 规则匹配 |
| 4 | 二进制文件 | 非支持的二进制扩展名 |
| 7 | PDF 页码格式错误 | 无效的 pages 参数 |
| 8 | PDF 页数超限 | 超过 20 页每次 |
| 9 | 设备文件阻断 | 阻断列表中的设备路径 |

---

## 7. 从零实现

### 7.1 核心架构

```
FileReadTool
├── inputSchema (file_path, offset, limit, pages)
├── outputSchema (text, image, notebook, pdf, parts, file_unchanged)
├── validateInput()      // 预读取检查
├── call()               // 主执行逻辑
│   ├── 缓存去重检查
│   ├── Skills 发现
│   └── callInner()      // 内部实现
│       ├── Notebook 处理
│       ├── Image 处理
│       ├── PDF 处理
│       └── Text 处理
├── mapToolResultToToolResultBlockParam()  // API 格式转换
└── UI 渲染函数
```

### 7.2 最小实现框架

```typescript
import { buildTool, type ToolDef } from '../../Tool.js'
import { z } from 'zod/v4'

// 输入 Schema
const inputSchema = z.strictObject({
  file_path: z.string().describe('The absolute path to the file to read'),
  offset: z.number().int().nonnegative().optional(),
  limit: z.number().int().positive().optional(),
  pages: z.string().optional(),
})

// 输出 Schema
const outputSchema = z.discriminatedUnion('type', [
  z.object({
    type: z.literal('text'),
    file: z.object({
      filePath: z.string(),
      content: z.string(),
      numLines: z.number(),
      startLine: z.number(),
      totalLines: z.number(),
    }),
  }),
  z.object({
    type: z.literal('image'),
    file: z.object({
      base64: z.string(),
      type: z.enum(['image/jpeg', 'image/png', 'image/gif', 'image/webp']),
      originalSize: z.number(),
    }),
  }),
  // ... 其他类型
])

export const FileReadTool = buildTool({
  name: 'Read',
  searchHint: 'read files, images, PDFs, notebooks',
  maxResultSizeChars: Infinity,
  strict: true,
  
  async description() {
    return 'Read a file from the local filesystem.'
  },
  
  get inputSchema() { return inputSchema },
  get outputSchema() { return outputSchema },
  
  isConcurrencySafe() { return true },
  isReadOnly() { return true },
  
  async validateInput({ file_path, pages }, context) {
    // 1. PDF 页码格式验证
    // 2. 权限 deny 规则检查
    // 3. UNC 路径安全检查
    // 4. 二进制扩展名检测
    // 5. 设备文件阻断检查
    return { result: true }
  },
  
  async call({ file_path, offset = 1, limit, pages }, context) {
    // 1. 获取读取限制配置
    // 2. 缓存去重检查
    // 3. Skills 发现
    // 4. 调用 callInner()
    // 5. ENOENT 处理 + 相似文件建议
    return { data: { type: 'text', file: { ... } } }
  },
  
  mapToolResultToToolResultBlockParam(data, toolUseID) {
    switch (data.type) {
      case 'text':
        return {
          tool_use_id: toolUseID,
          type: 'tool_result',
          content: addLineNumbers(data.file) + CYBER_RISK_MITIGATION_REMINDER,
        }
      case 'image':
        return {
          tool_use_id: toolUseID,
          type: 'tool_result',
          content: [{ type: 'image', source: { type: 'base64', ... } }],
        }
      // ...
    }
  },
})
```

### 7.3 文本读取核心逻辑

```typescript
async function callInner(...) {
  // Text file 处理
  const lineOffset = offset === 0 ? 0 : offset - 1
  const { content, lineCount, totalLines, totalBytes, readBytes, mtimeMs } =
    await readFileInRange(
      resolvedFilePath,
      lineOffset,
      limit,
      limit === undefined ? maxSizeBytes : undefined,
      context.abortController.signal,
    )

  await validateContentTokens(content, ext, maxTokens)

  // 更新缓存
  readFileState.set(fullFilePath, {
    content,
    timestamp: Math.floor(mtimeMs),
    offset,
    limit,
  })

  // 通知监听器
  for (const listener of fileReadListeners.slice()) {
    listener(resolvedFilePath, content)
  }

  return {
    data: {
      type: 'text',
      file: {
        filePath: file_path,
        content,
        numLines: lineCount,
        startLine: offset,
        totalLines,
      },
    },
  }
}
```

### 7.4 图像处理核心逻辑

```typescript
export async function readImageWithTokenBudget(
  filePath: string,
  maxTokens: number = getDefaultFileReadingLimits().maxTokens,
  maxBytes?: number,
): Promise<ImageResult> {
  // 读取文件一次
  const imageBuffer = await getFsImplementation().readFileBytes(filePath, maxBytes)
  const originalSize = imageBuffer.length

  // 检测格式
  const detectedMediaType = detectImageFormatFromBuffer(imageBuffer)
  const detectedFormat = detectedMediaType.split('/')[1] || 'png'

  // 标准缩放
  let result: ImageResult
  try {
    const resized = await maybeResizeAndDownsampleImageBuffer(
      imageBuffer,
      originalSize,
      detectedFormat,
    )
    result = createImageResponse(resized.buffer, resized.mediaType, originalSize, resized.dimensions)
  } catch (e) {
    // 失败时返回原图
    result = createImageResponse(imageBuffer, detectedFormat, originalSize)
  }

  // Token 预算检查
  const estimatedTokens = Math.ceil(result.file.base64.length * 0.125)
  if (estimatedTokens > maxTokens) {
    // 从同一 buffer 进行激进压缩
    const compressed = await compressImageBufferWithTokenLimit(
      imageBuffer,
      maxTokens,
      detectedMediaType,
    )
    return { type: 'image', file: { base64: compressed.base64, type: compressed.mediaType, originalSize } }
  }

  return result
}
```

### 7.5 行号格式化

**两种格式** (`utils/file.ts:278-319`)：

```typescript
export function isCompactLinePrefixEnabled(): boolean {
  // 通过 GB feature flag 控制
  return !getFeatureValue_CACHED_MAY_BE_STALE('tengu_compact_line_prefix_killswitch', false)
}

export function addLineNumbers({ content, startLine }: { content: string; startLine: number }): string {
  if (!content) return ''

  const lines = content.split(/\r?\n/)

  if (isCompactLinePrefixEnabled()) {
    // 紧凑格式：`N\t内容`
    return lines.map((line, index) => `${index + startLine}\t${line}`).join('\n')
  }

  // 传统格式：`     N→内容`
  return lines.map((line, index) => {
    const numStr = String(index + startLine)
    if (numStr.length >= 6) return `${numStr}→${line}`
    return `${numStr.padStart(6, ' ')}→${line}`
  }).join('\n')
}
```

---

## 8. 输出类型详解

### 8.1 Output Schema 完整定义

```typescript
const outputSchema = z.discriminatedUnion('type', [
  // 文本文件
  z.object({
    type: z.literal('text'),
    file: z.object({
      filePath: z.string(),
      content: z.string(),
      numLines: z.number(),
      startLine: z.number(),
      totalLines: z.number(),
    }),
  }),
  
  // 图像文件
  z.object({
    type: z.literal('image'),
    file: z.object({
      base64: z.string(),
      type: z.enum(['image/jpeg', 'image/png', 'image/gif', 'image/webp']),
      originalSize: z.number(),
      dimensions: z.object({
        originalWidth: z.number().optional(),
        originalHeight: z.number().optional(),
        displayWidth: z.number().optional(),
        displayHeight: z.number().optional(),
      }).optional(),
    }),
  }),
  
  // Jupyter Notebook
  z.object({
    type: z.literal('notebook'),
    file: z.object({
      filePath: z.string(),
      cells: z.array(z.any()),
    }),
  }),
  
  // PDF (整文档)
  z.object({
    type: z.literal('pdf'),
    file: z.object({
      filePath: z.string(),
      base64: z.string(),
      originalSize: z.number(),
    }),
  }),
  
  // PDF (分页提取)
  z.object({
    type: z.literal('parts'),
    file: z.object({
      filePath: z.string(),
      originalSize: z.number(),
      count: z.number(),
      outputDir: z.string(),
    }),
  }),
  
  // 缓存命中
  z.object({
    type: z.literal('file_unchanged'),
    file: z.object({
      filePath: z.string(),
    }),
  }),
])
```

### 8.2 UI 渲染逻辑

**工具使用消息** (`tools/FileReadTool/UI.tsx:30-65`)：
```typescript
export function renderToolUseMessage({ file_path, offset, limit, pages }, { verbose }) {
  if (!file_path) return null

  // Agent output 文件特殊处理
  if (getAgentOutputTaskId(file_path)) return ''

  const displayPath = verbose ? file_path : getDisplayPath(file_path)
  
  if (pages) {
    return <FilePathLink filePath={file_path}>{displayPath}</FilePathLink> + ` · pages ${pages}`
  }
  
  if (verbose && (offset || limit)) {
    const startLine = offset ?? 1
    const lineRange = limit 
      ? `lines ${startLine}-${startLine + limit - 1}` 
      : `from line ${startLine}`
    return <FilePathLink filePath={file_path}>{displayPath}</FilePathLink> + ` · ${lineRange}`
  }
  
  return <FilePathLink filePath={file_path}>{displayPath}</FilePathLink>
}
```

**工具结果消息** (`tools/FileReadTool/UI.tsx:77-143`)：
```typescript
export function renderToolResultMessage(output: Output) {
  switch (output.type) {
    case 'image':
      return <Text>Read image ({formatFileSize(output.file.originalSize)})</Text>
    case 'notebook':
      return <Text>Read <Text bold>{output.file.cells.length}</Text> cells</Text>
    case 'pdf':
      return <Text>Read PDF ({formatFileSize(output.file.originalSize)})</Text>
    case 'parts':
      return <Text>Read <Text bold>{output.file.count}</Text> pages ({formatFileSize(output.file.originalSize)})</Text>
    case 'text':
      return <Text>Read <Text bold>{output.file.numLines}</Text> lines</Text>
    case 'file_unchanged':
      return <Text dimColor>Unchanged since last read</Text>
  }
}
```

---

## 9. 读取限制配置

### 9.1 Limits 配置

**定义** (`tools/FileReadTool/limits.ts:35-40`)：
```typescript
export type FileReadingLimits = {
  maxTokens: number           // 最大 token 数
  maxSizeBytes: number        // 最大文件字节
  includeMaxSizeInPrompt?: boolean  // 是否在 prompt 中提示大小限制
  targetedRangeNudge?: boolean      // 是否启用精准范围提示
}

export const DEFAULT_MAX_OUTPUT_TOKENS = 25000
```

**配置来源优先级** (`tools/FileReadTool/limits.ts:53-92`)：
```
1. 环境变量 CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS
2. GrowthBook feature flag (tengu_amber_wren)
3. 默认值 DEFAULT_MAX_OUTPUT_TOKENS
```

**获取函数** (`tools/FileReadTool/limits.ts:24-33`)：
```typescript
function getEnvMaxTokens(): number | undefined {
  const override = process.env.CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS
  if (override) {
    const parsed = parseInt(override, 10)
    if (!isNaN(parsed) && parsed > 0) return parsed
  }
  return undefined
}
```

### 9.2 双重限制机制

| 限制 | 默认值 | 检查时机 | 检查对象 | 超限行为 |
|------|--------|---------|---------|---------|
| `maxSizeBytes` | 256KB | 预读取 | 文件总大小 | 抛出 FileTooLargeError |
| `maxTokens` | 25000 | 后读取 | 实际输出内容 | 抛出 MaxFileReadTokenExceededError |

**已知不匹配** (`tools/FileReadTool/limits.ts:9-14`)：
```
Known mismatch: maxSizeBytes gates on total file size, not the slice.
Tested truncating instead of throwing for explicit-limit reads that
exceed the byte cap (#21841, Mar 2026). Reverted: tool error rate
dropped but mean tokens rose — the throw path yields a ~100-byte error
tool-result while truncation yields ~25K tokens of content at the cap.
```

---

## 10. 监听器机制

### 10.1 文件读取监听器

**注册机制** (`tools/FileReadTool/FileReadTool.ts:161-173`)：
```typescript
type FileReadListener = (filePath: string, content: string) => void
const fileReadListeners: FileReadListener[] = []

export function registerFileReadListener(listener: FileReadListener): () => void {
  fileReadListeners.push(listener)
  return () => {
    const i = fileReadListeners.indexOf(listener)
    if (i >= 0) fileReadListeners.splice(i, 1)
  }
}
```

**调用时机** (`tools/FileReadTool/FileReadTool.ts:1041-1044`)：
```typescript
// Snapshot before iterating — a listener that unsubscribes mid-callback
// would splice the live array and skip the next listener.
for (const listener of fileReadListeners.slice()) {
  listener(resolvedFilePath, content)
}
```

---

## 11. 相关文件索引

| 文件路径 | 功能说明 |
|---------|---------|
| `tools/FileReadTool/FileReadTool.ts` | 主工具定义，核心逻辑 |
| `tools/FileReadTool/prompt.ts` | Prompt 模板渲染 |
| `tools/FileReadTool/UI.tsx` | UI 组件渲染 |
| `tools/FileReadTool/limits.ts` | 读取限制配置 |
| `tools/FileReadTool/imageProcessor.ts` | Sharp 图像处理模块加载 |
| `utils/readFileInRange.ts` | 大文件双路径读取策略 |
| `utils/imageResizer.ts` | 图像压缩、缩放逻辑 |
| `utils/file.ts` | 行号格式化、路径处理 |
| `services/tokenEstimation.ts` | Token 估算服务 |

---

*分析完成：2026-04-02*