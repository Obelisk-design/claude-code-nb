# FileEditTool 工具详解

**分析日期:** 2026-04-02

> **前置知识:** 阅读本文前，建议先阅读:
> - [03-工具系统设计](./03-工具系统设计.md) - 理解Tool接口与buildTool工厂

---

## 1. 概述

### 1.1 工具作用

FileEditTool 是 Claude Code CLI 中用于精确字符串替换编辑文件的核心工具。它的设计理念是：

**精确匹配替换** - 不同于传统编辑器的行号定位，FileEditTool 使用字符串匹配来定位编辑位置。这种方式更安全，因为 AI 可以直接看到文件内容并精确匹配要修改的文本。

**原子性操作** - 工具确保编辑操作的原子性，在读取文件状态后立即写入，防止并发修改导致的冲突。

**智能容错** - 支持引号样式规范化（弯引号 ↔ 直引号）、尾部空白处理、去规范化等智能匹配机制。

### 1.2 与传统编辑器的区别

| 特性 | 传统编辑器 | FileEditTool |
|------|-----------|--------------|
| 定位方式 | 行号 + 列号 | 精确字符串匹配 |
| 批量操作 | 手动逐个 | `replace_all` 参数一键替换所有匹配 |
| 安全检查 | 无强制 | 必须先读文件、检测并发修改 |
| 智能匹配 | 精确匹配 | 支持引号规范化、去规范化 |
| 冲突检测 | 无 | 自动检测多处匹配并提示 |
| 变更预览 | 实时预览 | Diff patch 结构化输出 |

**核心源码位置:**
- 主工具定义: `tools/FileEditTool/FileEditTool.ts`
- 类型定义: `tools/FileEditTool/types.ts`
- 工具函数: `tools/FileEditTool/utils.ts`
- UI渲染: `tools/FileEditTool/UI.tsx`
- 工具描述: `tools/FileEditTool/prompt.ts`
- 常量定义: `tools/FileEditTool/constants.ts`

---

## 2. 输入 Schema

### 2.1 参数定义

```typescript
// tools/FileEditTool/types.ts
const inputSchema = z.strictObject({
  file_path: z.string().describe('The absolute path to the file to modify'),
  old_string: z.string().describe('The text to replace'),
  new_string: z.string().describe(
    'The text to replace it with (must be different from old_string)'
  ),
  replace_all: semanticBoolean(
    z.boolean().default(false).optional()
  ).describe('Replace all occurrences of old_string (default false)'),
})
```

### 2.2 file_path 参数

**要求:** 必须是绝对路径

**路径处理:**
```typescript
// FileEditTool.ts: 第118-120行
backfillObservableInput(input) {
  if (typeof input.file_path === 'string') {
    input.file_path = expandPath(input.file_path)  // 展开 ~ 和相对路径
  }
}
```

**支持的路径格式:**
- 绝对路径: `/home/user/file.txt`, `D:\project\file.ts`
- Home路径: `~/project/file.txt` → 自动展开
- 相对路径: 会被 `expandPath()` 转换为绝对路径

**安全检查:** UNC路径 (`\\server\share`) 被特殊处理以防止 NTLM 凭据泄露:
```typescript
// FileEditTool.ts: 第179-181行
if (fullFilePath.startsWith('\\\\') || fullFilePath.startsWith('//')) {
  return { result: true }  // 跳过文件系统操作
}
```

### 2.3 old_string / new_string 参数

**核心规则:**

1. **old_string 必须唯一** - 如果文件中有多个匹配且 `replace_all=false`，工具会拒绝操作并报错
2. **old_string 和 new_string 必须不同** - 相同则无意义，工具会拒绝
3. **精确匹配** - 包括缩进、空格、换行符在内的所有字符必须完全匹配
4. **空字符串特殊含义** - `old_string=""` 表示创建新文件

**字符串长度统计:**
```typescript
// FileEditTool.ts: 第539-543行
logEvent('tengu_edit_string_lengths', {
  oldStringBytes: Buffer.byteLength(old_string, 'utf8'),
  newStringBytes: Buffer.byteLength(new_string, 'utf8'),
  replaceAll: replace_all,
})
```

### 2.4 replace_all 参数

**默认值:** `false`

**用途:** 当文件中存在多个相同的 `old_string` 匹配时，设置为 `true` 可以一次性替换所有匹配。

**典型场景:**
- 变量重命名: 将所有 `oldVarName` 替换为 `newVarName`
- API 端点更新: 替换所有旧 URL
- 函数名称修改: 批量更新所有调用处

---

## 3. 编辑算法详解

### 3.1 字符串匹配算法

**多层级匹配策略:**

```typescript
// tools/FileEditTool/utils.ts: 第73-93行
export function findActualString(
  fileContent: string,
  searchString: string,
): string | null {
  // 第一层: 精确匹配
  if (fileContent.includes(searchString)) {
    return searchString
  }

  // 第二层: 引号规范化匹配
  const normalizedSearch = normalizeQuotes(searchString)
  const normalizedFile = normalizeQuotes(fileContent)

  const searchIndex = normalizedFile.indexOf(normalizedSearch)
  if (searchIndex !== -1) {
    return fileContent.substring(searchIndex, searchIndex + searchString.length)
  }

  return null
}
```

**引号规范化处理:**
```typescript
// tools/FileEditTool/utils.ts: 第31-37行
export function normalizeQuotes(str: string): string {
  return str
    .replaceAll(LEFT_SINGLE_CURLY_QUOTE, "'")   // ' → '
    .replaceAll(RIGHT_SINGLE_CURLY_QUOTE, "'")   // ' → '
    .replaceAll(LEFT_DOUBLE_CURLY_QUOTE, '"')    // " → "
    .replaceAll(RIGHT_DOUBLE_CURLY_QUOTE, '"')   // " → "
}
```

**支持的弯引号字符:**
```typescript
// tools/FileEditTool/utils.ts: 第21-24行
export const LEFT_SINGLE_CURLY_QUOTE = ''   // U+2018
export const RIGHT_SINGLE_CURLY_QUOTE = ''  // U+2019
export const LEFT_DOUBLE_CURLY_QUOTE = '"'   // U+201C
export const RIGHT_DOUBLE_CURLY_QUOTE = '"'  // U+201D
```

### 3.2 多处替换处理

**冲突检测逻辑:**

```typescript
// FileEditTool.ts: 第329-343行
const matches = file.split(actualOldString).length - 1

if (matches > 1 && !replace_all) {
  return {
    result: false,
    behavior: 'ask',
    message: `Found ${matches} matches of the string to replace, but replace_all is false. To replace all occurrences, set replace_all to true. To replace only one occurrence, please provide more context to uniquely identify the instance.\nString: ${old_string}`,
    errorCode: 9,
  }
}
```

**用户选择:**
- 设置 `replace_all=true` 替换所有匹配
- 扩大 `old_string` 范围添加更多上下文使其唯一

### 3.3 编辑冲突检测

**并发修改检测:**

```typescript
// FileEditTool.ts: 第290-310行
const readTimestamp = readFileState.get(fullFilePath)

if (readTimestamp) {
  const lastWriteTime = getFileModificationTime(fullFilePath)
  if (lastWriteTime > readTimestamp.timestamp) {
    // Windows 时间戳可能因云同步/杀毒软件变化而无实际修改
    // 全量读取时比较内容作为后备方案
    const isFullRead = 
      readTimestamp.offset === undefined && 
      readTimestamp.limit === undefined
    if (isFullRead && fileContent === readTimestamp.content) {
      // 内容未变，安全继续
    } else {
      return {
        result: false,
        message: 'File has been modified since read...',
        errorCode: 7,
      }
    }
  }
}
```

**原子性保证:**

```typescript
// FileEditTool.ts: 第442-468行
// 关键区: 从读取到写入之间避免异步操作
const {
  content: originalFileContents,
  fileExists,
  encoding,
  lineEndings: endings,
} = readFileForEdit(absoluteFilePath)

if (fileExists) {
  const lastWriteTime = getFileModificationTime(absoluteFilePath)
  const lastRead = readFileState.get(absoluteFilePath)
  if (!lastRead || lastWriteTime > lastRead.timestamp) {
    // 检查冲突...
    if (!contentUnchanged) {
      throw new Error(FILE_UNEXPECTEDLY_MODIFIED_ERROR)
    }
  }
}

// 立即写入，中间无异步操作
writeTextContent(absoluteFilePath, updatedFile, encoding, endings)
```

### 3.4 Patch 生成

**Diff 算法:**

```typescript
// tools/FileEditTool/utils.ts: 第234-254行
export function getPatchForEdit({
  filePath,
  fileContents,
  oldString,
  newString,
  replaceAll = false,
}): { patch: StructuredPatchHunk[]; updatedFile: string } {
  return getPatchForEdits({
    filePath,
    fileContents,
    edits: [
      { old_string: oldString, new_string: newString, replace_all: replaceAll },
    ],
  })
}

// 第262-350行
export function getPatchForEdits({
  filePath,
  fileContents,
  edits,
}): { patch: StructuredPatchHunk[]; updatedFile: string } {
  let updatedFile = fileContents
  const appliedNewStrings: string[] = []

  for (const edit of edits) {
    // 检查 old_string 是否是之前 new_string 的子串
    for (const previousNewString of appliedNewStrings) {
      if (previousNewString.includes(oldStringToCheck)) {
        throw new Error('Cannot edit file: old_string is substring of new_string from previous edit.')
      }
    }

    // 应用编辑
    updatedFile = applyEditToFile(updatedFile, edit.old_string, edit.new_string, edit.replace_all)
    
    // 检查是否实际发生了变化
    if (updatedFile === previousContent) {
      throw new Error('String not found in file.')
    }
    
    appliedNewStrings.push(edit.new_string)
  }

  // 使用 diff 库生成结构化 patch
  const patch = getPatchFromContents({
    filePath,
    oldContent: convertLeadingTabsToSpaces(fileContents),
    newContent: convertLeadingTabsToSpaces(updatedFile),
  })

  return { patch, updatedFile }
}
```

### 3.5 引号样式保留

当文件使用弯引号但 AI 输出直引号时，工具会自动保留文件的引号样式:

```typescript
// tools/FileEditTool/utils.ts: 第104-136行
export function preserveQuoteStyle(
  oldString: string,
  actualOldString: string,
  newString: string,
): string {
  if (oldString === actualOldString) {
    return newString  // 无规范化发生
  }

  // 检测文件中的弯引号类型
  const hasDoubleQuotes = 
    actualOldString.includes(LEFT_DOUBLE_CURLY_QUOTE) ||
    actualOldString.includes(RIGHT_DOUBLE_CURLY_QUOTE)
  const hasSingleQuotes = 
    actualOldString.includes(LEFT_SINGLE_CURLY_QUOTE) ||
    actualOldString.includes(RIGHT_SINGLE_CURLY_QUOTE)

  let result = newString
  if (hasDoubleQuotes) result = applyCurlyDoubleQuotes(result)
  if (hasSingleQuotes) result = applyCurlySingleQuotes(result)

  return result
}
```

**智能区分引号和撇号:**

```typescript
// tools/FileEditTool/utils.ts: 第173-199行
function applyCurlySingleQuotes(str: string): string {
  const chars = [...str]
  for (let i = 0; i < chars.length; i++) {
    if (chars[i] === "'") {
      const prevIsLetter = prev !== undefined && /\p{L}/u.test(prev)
      const nextIsLetter = next !== undefined && /\p{L}/u.test(next)
      
      if (prevIsLetter && nextIsLetter) {
        // 缩写中的撇号，如 "don't"
        result.push(RIGHT_SINGLE_CURLY_QUOTE)
      } else {
        // 真正的引号
        result.push(isOpeningContext(chars, i) ? LEFT : RIGHT)
      }
    }
  }
  return result.join('')
}
```

---

## 4. 用户使用指南

### 4.1 如何精确匹配

**核心原则: 从 Read 工具输出复制，保持格式不变**

Read 工具的输出格式:
```
     1→const x = 1
     2→function test() {
     3→  return x
     4→}
```

**正确做法:**
```typescript
old_string: "function test() {\n  return x\n}"
```

**错误做法:**
```typescript
// 错误1: 包含行号前缀
old_string: "     2→function test() {\n     3→  return x\n     4→}"

// 错误2: 缩进不匹配（Read显示的是转换后的空格，实际文件可能是Tab）
// 需要注意: Read工具会将Tab转换为空格显示，但Edit需要匹配原始文件
```

**行号前缀格式:**
```typescript
// tools/FileEditTool/prompt.ts: 第13-15行
const prefixFormat = isCompactLinePrefixEnabled()
  ? 'line number + tab'         // 紧凑模式: "1\t"
  : 'spaces + line number + arrow'  // 默认模式: "     1→"
```

**工具提示:**
```typescript
// tools/FileEditTool/prompt.ts: 第22-23行
"When editing text from Read tool output, ensure you preserve the exact indentation 
(tab/spaces) as it appears AFTER the line number prefix."
```

### 4.2 批量替换技巧

**场景: 重命名变量**

```typescript
// 单次替换一处
{
  file_path: "/project/src/app.ts",
  old_string: "userName",
  new_string: "username",
  replace_all: false  // 仅替换第一个匹配
}

// 批量替换所有
{
  file_path: "/project/src/app.ts",
  old_string: "userName",
  new_string: "username",
  replace_all: true  // 替换所有匹配
}
```

**最佳实践: 使用最小唯一匹配**

```typescript
// tools/FileEditTool/prompt.ts: 第16-19行
const minimalUniquenessHint =
  process.env.USER_TYPE === 'ant'
    ? "Use the smallest old_string that's clearly unique — usually 2-4 adjacent 
       lines is sufficient. Avoid including 10+ lines of context when less 
       uniquely identifies the target."
    : ''
```

**建议:**
- 2-4 行上下文通常足够唯一标识
- 不要包含 10+ 行的不必要上下文
- 包含函数签名或变量声明即可

### 4.3 编辑失败排查

**错误码对照表:**

| errorCode | 含义 | 解决方案 |
|-----------|------|----------|
| 0 | 包含敏感信息 | 检查 new_string 是否泄露密钥 |
| 1 | old_string === new_string | 确保字符串确实不同 |
| 2 | 权限拒绝 | 检查 permission settings |
| 3 | 文件已存在但试图创建 | 使用非空 old_string 编辑 |
| 4 | 文件不存在 | 检查路径是否正确 |
| 5 | .ipynb 文件 | 使用 NotebookEditTool |
| 6 | 未先读取文件 | 先调用 Read 工具 |
| 7 | 文件并发修改 | 再次读取文件 |
| 8 | 字符串未找到 | 检查精确匹配 |
| 9 | 多处匹配但 replace_all=false | 设置 replace_all 或扩大上下文 |
| 10 | 文件过大 (>1GB) | 使用其他方式处理 |

**常见错误及解决:**

**错误1: "String to replace not found in file"**
```
原因: old_string 与文件内容不完全匹配
排查:
1. 检查缩进（Tab vs 空格）
2. 检查换行符（LF vs CRLF）
3. 检查行尾空白
4. 使用 Read 工具重新查看文件
```

**错误2: "Found N matches but replace_all is false"**
```
解决:
- 方案A: 设置 replace_all: true
- 方案B: 扩大 old_string 范围使其唯一
```

**错误3: "File has not been read yet"**
```
解决: 在编辑前调用 Read 工具
```

**错误4: "File has been modified since read"**
```
解决: 再次读取文件以获取最新内容
```

### 4.4 最佳实践

**1. 先读后写原则**

```typescript
// tools/FileEditTool/prompt.ts: 第4-6行
function getPreReadInstruction(): string {
  return `\n- You must use your \`${FILE_READ_TOOL_NAME}\` tool at least once 
           in the conversation before editing. This tool will error if you 
           attempt an edit without reading the file.`
}
```

**2. 编辑优于新建**

```typescript
// tools/FileEditTool/prompt.ts: 第24行
"ALWAYS prefer editing existing files in the codebase. NEVER write new files 
 unless explicitly required."
```

**3. 路径使用绝对路径**

```typescript
// tools/FileEditTool/FileEditTool.ts: 第112-114行
getPath(input): string {
  return input.file_path  // 返回路径用于权限检查
}
```

**4. 大文件限制**

```typescript
// FileEditTool.ts: 第84行
const MAX_EDIT_FILE_SIZE = 1024 * 1024 * 1024  // 1 GiB

// 第186-195行
const { size } = await fs.stat(fullFilePath)
if (size > MAX_EDIT_FILE_SIZE) {
  return {
    result: false,
    message: `File is too large to edit (${formatFileSize(size)}). 
               Maximum editable file size is ${formatFileSize(MAX_EDIT_FILE_SIZE)}.`,
  }
}
```

**5. Markdown 文件特殊处理**

```typescript
// tools/FileEditTool/utils.ts: 第597行
const isMarkdown = /\.(md|mdx)$/i.test(file_path)

// Markdown 使用两个尾部空格表示硬换行
// 工具会跳过 Markdown 文件的尾部空白处理
```

---

## 5. 与 FileReadTool 配合

### 5.1 读取状态追踪

FileEditTool 维护 `readFileState` 来追踪已读取的文件:

```typescript
// FileEditTool.ts: 第275-287行
const readTimestamp = readFileState.get(fullFilePath)

if (!readTimestamp || readTimestamp.isPartialView) {
  return {
    result: false,
    message: 'File has not been read yet. Read it first before writing to it.',
  }
}
```

**状态记录:**
```typescript
// FileEditTool.ts: 第520-525行
readFileState.set(absoluteFilePath, {
  content: updatedFile,
  timestamp: getFileModificationTime(absoluteFilePath),
  offset: undefined,  // 全量读取标记
  limit: undefined,
})
```

### 5.2 工作流程

```
1. Read Tool → 读取文件内容
   ↓
2. readFileState 记录: {content, timestamp, offset, limit}
   ↓
3. AI 分析内容，生成 old_string/new_string
   ↓
4. Edit Tool → 检查 readFileState
   ↓
5. 验证时间戳 → 确认无并发修改
   ↓
6. 应用编辑 → writeTextContent
   ↓
7. 更新 readFileState → 新时间戳
```

### 5.3 部分读取处理

```typescript
// FileEditTool.ts: 第276行
if (!readTimestamp || readTimestamp.isPartialView) {
  // 部分读取（offset/limit 有值）需要重新全量读取
}
```

**部分读取风险:** 如果只读取了文件的某一部分，编辑可能影响未读取区域，工具会拒绝操作。

---

## 6. 安全检查

### 6.1 路径验证

**路径展开:**
```typescript
// FileEditTool.ts: 第118-120行
backfillObservableInput(input) {
  if (typeof input.file_path === 'string') {
    input.file_path = expandPath(input.file_path)
  }
}
```

**绝对路径检查:**
```typescript
// FileEditTool.ts: 第283-285行
meta: {
  isFilePathAbsolute: String(isAbsolute(file_path)),
}
```

**UNC 路径安全处理:**
```typescript
// FileEditTool.ts: 第176-181行
// 安全: 跳过 UNC 路径的文件系统操作防止 NTLM 凭据泄露
if (fullFilePath.startsWith('\\\\') || fullFilePath.startsWith('//')) {
  return { result: true }
}
```

### 6.2 权限检查

**写入权限验证:**
```typescript
// FileEditTool.ts: 第125-131行
async checkPermissions(input, context): Promise<PermissionDecision> {
  const appState = context.getAppState()
  return checkWritePermissionForTool(
    FileEditTool,
    input,
    appState.toolPermissionContext,
  )
}
```

**权限规则匹配:**
```typescript
// FileEditTool.ts: 第159-174行
const denyRule = matchingRuleForInput(
  fullFilePath,
  appState.toolPermissionContext,
  'edit',
  'deny',
)

if (denyRule !== null) {
  return {
    result: false,
    message: 'File is in a directory that is denied by your permission settings.',
  }
}
```

**通配符模式匹配:**
```typescript
// FileEditTool.ts: 第122-124行
async preparePermissionMatcher({ file_path }) {
  return pattern => matchWildcardPattern(pattern, file_path)
}
```

### 6.3 敏感信息检测

**Team Memory 密钥检测:**
```typescript
// FileEditTool.ts: 第143-147行
const secretError = checkTeamMemSecrets(fullFilePath, new_string)
if (secretError) {
  return { result: false, message: secretError, errorCode: 0 }
}
```

### 6.4 Settings 文件验证

```typescript
// FileEditTool.ts: 第345-359行
const settingsValidationResult = validateInputForSettingsFileEdit(
  fullFilePath,
  file,
  () => {
    // 模拟编辑获取最终内容
    return replace_all
      ? file.replaceAll(actualOldString, new_string)
      : file.replace(actualOldString, new_string)
  },
)

if (settingsValidationResult !== null) {
  return settingsValidationResult
}
```

### 6.5 文件存在性检查

**不存在文件处理:**
```typescript
// FileEditTool.ts: 第224-246行
if (fileContent === null) {
  if (old_string === '') {
    return { result: true }  // 创建新文件
  }
  
  // 查找相似文件名建议
  const similarFilename = findSimilarFile(fullFilePath)
  const cwdSuggestion = await suggestPathUnderCwd(fullFilePath)
  
  let message = `File does not exist. ${FILE_NOT_FOUND_CWD_NOTE} ${getCwd()}.`
  if (cwdSuggestion) message += ` Did you mean ${cwdSuggestion}?`
  else if (similarFilename) message += ` Did you mean ${similarFilename}?`
  
  return { result: false, message }
}
```

---

## 7. 从零实现

### 7.1 核心结构

```typescript
// 最小实现框架
interface FileEditInput {
  file_path: string
  old_string: string
  new_string: string
  replace_all?: boolean
}

interface FileEditOutput {
  filePath: string
  oldString: string
  newString: string
  originalFile: string
  structuredPatch: DiffHunk[]
  userModified: boolean
  replaceAll: boolean
}

class FileEditTool {
  name = 'Edit'
  
  // 核心流程
  async call(input: FileEditInput, context: ToolContext): Promise<FileEditOutput> {
    // 1. 验证输入
    const validation = await this.validateInput(input, context)
    if (!validation.result) throw new Error(validation.message)
    
    // 2. 读取文件
    const originalContent = await this.readFile(input.file_path)
    
    // 3. 查找匹配
    const actualOldString = this.findActualString(originalContent, input.old_string)
    if (!actualOldString) throw new Error('String not found')
    
    // 4. 检查多处匹配
    const matches = originalContent.split(actualOldString).length - 1
    if (matches > 1 && !input.replace_all) {
      throw new Error(`Found ${matches} matches, set replace_all=true or add context`)
    }
    
    // 5. 应用替换
    const updatedContent = input.replace_all
      ? originalContent.replaceAll(actualOldString, input.new_string)
      : originalContent.replace(actualOldString, input.new_string)
    
    // 6. 生成 patch
    const patch = this.generatePatch(originalContent, updatedContent)
    
    // 7. 写入文件
    await this.writeFile(input.file_path, updatedContent)
    
    // 8. 返回结果
    return {
      filePath: input.file_path,
      oldString: actualOldString,
      newString: input.new_string,
      originalFile: originalContent,
      structuredPatch: patch,
      userModified: false,
      replaceAll: input.replace_all ?? false,
    }
  }
}
```

### 7.2 字符串匹配实现

```typescript
function findActualString(fileContent: string, searchString: string): string | null {
  // 精确匹配
  if (fileContent.includes(searchString)) {
    return searchString
  }
  
  // 引号规范化匹配
  const curlyQuotes = {
    ''': "'",  ''': "'",  '"': '"', '"': '"'
  }
  
  let normalizedSearch = searchString
  let normalizedFile = fileContent
  
  for (const [curly, straight] of Object.entries(curlyQuotes)) {
    normalizedSearch = normalizedSearch.replaceAll(curly, straight)
    normalizedFile = normalizedFile.replaceAll(curly, straight)
  }
  
  const index = normalizedFile.indexOf(normalizedSearch)
  if (index !== -1) {
    return fileContent.substring(index, index + searchString.length)
  }
  
  return null
}
```

### 7.3 冲突检测实现

```typescript
async function checkConcurrency(filePath: string, readState: ReadFileState): boolean {
  const lastWriteTime = fs.statSync(filePath).mtimeMs
  const lastRead = readState.get(filePath)
  
  if (!lastRead) return false  // 未读取
  
  if (lastWriteTime > lastRead.timestamp) {
    // 时间戳变化，检查内容是否实际改变（Windows后备方案）
    const currentContent = fs.readFileSync(filePath, 'utf8')
    if (currentContent === lastRead.content) {
      return true  // 内容未变
    }
    return false  // 确实被修改
  }
  
  return true  // 无冲突
}
```

### 7.4 Patch 生成实现

```typescript
function generatePatch(oldContent: string, newContent: string): DiffHunk[] {
  // 使用 diff 库
  const { structuredPatch } = require('diff')
  
  const patch = structuredPatch(
    'file.txt',
    'file.txt',
    oldContent,
    newContent,
    undefined,
    undefined,
    { context: 4 }
  )
  
  return patch.hunks.map(hunk => ({
    oldStart: hunk.oldStart,
    oldLines: hunk.oldLines,
    newStart: hunk.newStart,
    newLines: hunk.newLines,
    lines: hunk.lines,
  }))
}
```

### 7.5 完整工具定义

```typescript
const FileEditTool = {
  name: 'Edit',
  inputSchema: z.object({
    file_path: z.string(),
    old_string: z.string(),
    new_string: z.string(),
    replace_all: z.boolean().optional().default(false),
  }),
  
  outputSchema: z.object({
    filePath: z.string(),
    oldString: z.string(),
    newString: z.string(),
    originalFile: z.string(),
    structuredPatch: z.array(z.any()),
    userModified: z.boolean(),
    replaceAll: z.boolean(),
  }),
  
  validateInput: async (input, context) => {
    // 1. 检查字符串不同
    if (input.old_string === input.new_string) {
      return { result: false, message: 'Strings are identical' }
    }
    
    // 2. 检查路径权限
    // 3. 检查文件大小
    // 4. 检查文件存在性
    // 5. 检查读取状态
    // 6. 检查并发修改
    
    return { result: true }
  },
  
  call: async (input, context) => {
    // 实现核心流程
  },
}
```

---

## 附录: 输出格式示例

### 成功编辑输出

```typescript
{
  filePath: "/project/src/app.ts",
  oldString: "function oldName() {",
  newString: "function newName() {",
  originalFile: "...",
  structuredPatch: [
    {
      oldStart: 10,
      oldLines: 3,
      newStart: 10,
      newLines: 3,
      lines: [" function oldName() {", "-  return 1", "+  return 2", " }"]
    }
  ],
  userModified: false,
  replaceAll: false,
}
```

### 工具结果消息

```typescript
// FileEditTool.ts: 第575-594行
mapToolResultToToolResultBlockParam(data, toolUseID) {
  const modifiedNote = userModified
    ? '. The user modified your proposed changes before accepting them.'
    : ''

  if (replaceAll) {
    return {
      type: 'tool_result',
      content: `The file ${filePath} has been updated${modifiedNote}. 
                All occurrences were successfully replaced.`
    }
  }

  return {
    type: 'tool_result',
    content: `The file ${filePath} has been updated successfully${modifiedNote}.`
  }
}
```

---

## 总结

FileEditTool 是 Claude Code CLI 的核心编辑工具，其设计精髓在于:

1. **精确字符串匹配** - 安全可靠的定位方式
2. **智能容错** - 引号规范化、去规范化、尾部空白处理
3. **原子性保证** - 读取状态追踪、并发修改检测
4. **完整安全检查** - 权限、敏感信息、文件大小限制
5. **结构化输出** - Diff patch 用于用户预览和确认

掌握 FileEditTool 的关键是理解 "先读后写" 原则和精确匹配要求。通过本教程，您应该能够:

- 正确构造编辑参数
- 处理常见错误情况
- 理解内部工作原理
- 实现类似功能

**源码目录:** `tools/FileEditTool/`
**核心文件:** `FileEditTool.ts` (626 行)