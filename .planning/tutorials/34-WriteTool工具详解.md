# Write工具详解

**分析日期:** 2026-04-02

## 1. 概述

### 1.1 Write工具的作用

Write工具（FileWriteTool）是Claude Code CLI中负责**文件写入**的核心工具。它的主要职责是：

- **创建新文件**：向指定路径写入全新内容
- **覆盖已有文件**：用新内容完全替换现有文件内容
- **原子性写入**：确保写入操作的完整性和一致性
- **权限验证**：检查是否允许写入指定路径
- **状态追踪**：记录文件读取状态，防止并发写入冲突

工具定义位于 `tools/FileWriteTool/FileWriteTool.ts`，导出名称为 `Write`。

### 1.2 与FileEditTool（Edit工具）的区别

| 特性 | Write工具 | Edit工具 |
|------|-----------|----------|
| **操作方式** | 完整内容替换 | 差异化编辑（diff） |
| **适用场景** | 创建新文件、完全重写 | 修改现有文件的部分内容 |
| **数据传输** | 传输完整文件内容 | 仅传输修改的diff片段 |
| **效率** | 大文件时效率较低 | 小改动时效率高 |
| **强制要求** | 已存在文件必须先Read | 已存在文件必须先Read |
| **输出显示** | 新文件显示前10行，更新显示diff | 显示变更diff |

**核心区别：**
- Write工具是"全量替换"，适合创建新文件或需要完全重写的场景
- Edit工具是"增量修改"，适合对现有文件做小范围修改的场景

源码中的描述明确指出：
```typescript
// prompt.ts 第15行
- Prefer the Edit tool for modifying existing files — it only sends the diff. 
  Only use this tool to create new files or for complete rewrites.
```

---

## 2. 输入Schema

### 2.1 输入参数定义

Write工具的输入Schema定义在 `FileWriteTool.ts` 第56-65行：

```typescript
const inputSchema = lazySchema(() =>
  z.strictObject({
    file_path: z
      .string()
      .describe(
        'The absolute path to the file to write (must be absolute, not relative)',
      ),
    content: z.string().describe('The content to write to the file'),
  }),
)
```

### 2.2 file_path参数

**类型：** `string`

**要求：**
- 必须是**绝对路径**，不能是相对路径
- 支持路径扩展（如 `~` 展开为用户目录）
- 支持环境变量替换

**路径处理流程：**
```typescript
// FileWriteTool.ts 第128-130行
backfillObservableInput(input) {
  if (typeof input.file_path === 'string') {
    input.file_path = expandPath(input.file_path)  // 展开路径
  }
}
```

`expandPath` 函数处理以下转换：
- `~` → 用户主目录
- `$HOME` → 用户主目录
- 相对路径 → 基于当前工作目录的绝对路径

### 2.3 content参数

**类型：** `string`

**要求：**
- 文件的完整内容
- 会保留用户提供的换行符格式（LF/CRLF）
- 内容直接写入，不做额外转换

**换行符处理说明（第300-305行）：**
```typescript
// Write is a full content replacement — the model sent explicit line endings
// in `content` and meant them. Do not rewrite them. Previously we preserved
// the old file's line endings (or sampled the repo via ripgrep for new
// files), which silently corrupted e.g. bash scripts with \r on Linux when
// overwriting a CRLF file or when binaries in cwd poisoned the repo sample.
writeTextContent(fullFilePath, content, enc, 'LF')
```

---

## 3. 写入实现

### 3.1 文件创建流程

当目标文件不存在时，Write工具执行创建流程：

**关键代码（第268-277行）：**
```typescript
let meta: ReturnType<typeof readFileSyncWithMetadata> | null
try {
  meta = readFileSyncWithMetadata(fullFilePath)
} catch (e) {
  if (isENOENT(e)) {
    meta = null  // 文件不存在，标记为null
  } else {
    throw e
  }
}
```

**创建新文件时：**
1. 尝试读取文件元数据，捕获 `ENOENT`（文件不存在）错误
2. `meta = null` 表示这是创建操作
3. 确保父目录存在（第254行）
4. 直接写入内容
5. 返回 `type: 'create'` 的结果

**返回数据结构（第395-402行）：**
```typescript
const data = {
  type: 'create' as const,
  filePath: file_path,
  content,
  structuredPatch: [],       // 新文件无diff
  originalFile: null,        // 新文件无原始内容
  ...(gitDiff && { gitDiff }),
}
```

### 3.2 覆盖检查机制

Write工具有**严格的覆盖保护机制**，防止并发写入冲突。

**检查流程（第153-221行 validateInput函数）：**

```typescript
async validateInput({ file_path, content }, toolUseContext: ToolUseContext) {
  const fullFilePath = expandPath(file_path)

  // 1. 检查是否包含敏感信息（team memory secrets）
  const secretError = checkTeamMemSecrets(fullFilePath, content)
  if (secretError) {
    return { result: false, message: secretError, errorCode: 0 }
  }

  // 2. 检查权限拒绝规则
  const denyRule = matchingRuleForInput(fullFilePath, appState.toolPermissionContext, 'edit', 'deny')
  if (denyRule !== null) {
    return { result: false, message: 'File is in a directory that is denied...', errorCode: 1 }
  }

  // 3. 安全检查：跳过UNC路径的文件系统操作（防止NTLM凭据泄露）
  if (fullFilePath.startsWith('\\\\') || fullFilePath.startsWith('//')) {
    return { result: true }
  }

  // 4. 获取文件修改时间
  const fileStat = await fs.stat(fullFilePath)
  fileMtimeMs = fileStat.mtimeMs

  // 5. 检查是否已读取文件
  const readTimestamp = toolUseContext.readFileState.get(fullFilePath)
  if (!readTimestamp || readTimestamp.isPartialView) {
    return { result: false, message: 'File has not been read yet...', errorCode: 2 }
  }

  // 6. 检查文件是否在读取后被修改
  const lastWriteTime = Math.floor(fileMtimeMs)
  if (lastWriteTime > readTimestamp.timestamp) {
    return { result: false, message: 'File has been modified since read...', errorCode: 3 }
  }

  return { result: true }
}
```

**错误码含义：**

| errorCode | 含义 |
|-----------|------|
| 0 | Team memory secrets检查失败 |
| 1 | 权限拒绝规则匹配 |
| 2 | 文件未读取或部分读取 |
| 3 | 文件在读取后被修改 |

### 3.3 原子写入实现

Write工具实现了**原子性写入**，确保在并发环境下的一致性。

**原子写入的关键代码（第266-295行）：**

```typescript
// Load current state and confirm no changes since last read.
// Please avoid async operations between here and writing to disk 
// to preserve atomicity.
let meta: ReturnType<typeof readFileSyncWithMetadata> | null
try {
  meta = readFileSyncWithMetadata(fullFilePath)
} catch (e) {
  if (isENOENT(e)) {
    meta = null
  } else {
    throw e
  }
}

if (meta !== null) {
  const lastWriteTime = getFileModificationTime(fullFilePath)
  const lastRead = readFileState.get(fullFilePath)
  if (!lastRead || lastWriteTime > lastRead.timestamp) {
    // Windows时间戳可能因云同步/杀毒软件变化，进行内容比对
    const isFullRead = lastRead && lastRead.offset === undefined && lastRead.limit === undefined
    if (!isFullRead || meta.content !== lastRead.content) {
      throw new Error(FILE_UNEXPECTEDLY_MODIFIED_ERROR)
    }
  }
}

// 直接写入，无异步操作间隔
writeTextContent(fullFilePath, content, enc, 'LF')
```

**原子写入要点：**
1. **读取-验证-写入**在临界区内连续执行
2. **避免异步操作间隔**（注释明确警告）
3. **双重验证**：时间戳 + 内容比对（针对Windows平台）
4. **写入后立即更新读取状态**

**写入函数实现（`utils/file.ts` 第84-98行）：**
```typescript
export function writeTextContent(
  filePath: string,
  content: string,
  encoding: BufferEncoding,
  endings: LineEndingType,
): void {
  let toWrite = content
  if (endings === 'CRLF') {
    // 先规范化CRLF为LF，再转换为CRLF
    toWrite = content.replaceAll('\r\n', '\n').split('\n').join('\r\n')
  }
  writeFileSyncAndFlush_DEPRECATED(filePath, toWrite, { encoding })
}
```

---

## 4. 用户使用指南

### 4.1 何时使用Write vs Edit

**使用Write工具的场景：**
- 创建全新的文件
- 需要完全重写文件内容（如配置文件重构）
- 文件内容大部分需要变更
- 需要确保文件最终状态的确定性

**使用Edit工具的场景：**
- 修改文件的部分内容
- 添加或删除几行代码
- 替换特定文本片段
- 小范围的代码调整

**工具提示中的建议（`prompt.ts` 第14-17行）：**
```typescript
- Prefer the Edit tool for modifying existing files — it only sends the diff. 
  Only use this tool to create new files or for complete rewrites.
- NEVER create documentation files (*.md) or README files unless explicitly 
  requested by the User.
- Only use emojis if the user explicitly requests it. Avoid writing emojis 
  to files unless asked.
```

### 4.2 创建新文件

**步骤：**

1. **确定绝对路径**
   ```
   file_path: "/project/src/components/NewComponent.tsx"
   ```

2. **提供完整内容**
   ```
   content: "import React from 'react';\n\nexport function NewComponent() {\n  return <div>New Component</div>;\n}"
   ```

3. **Write工具自动：**
   - 检查父目录是否存在，不存在则创建
   - 验证权限
   - 写入内容
   - 记录文件状态

**目录自动创建（第254行）：**
```typescript
await getFsImplementation().mkdir(dir)
```

### 4.3 安全写入已有文件

**强制要求：必须先使用Read工具读取文件**

原因：
1. 防止覆盖未预览的内容
2. 记录读取时间戳，用于并发冲突检测
3. 确保AI了解当前文件状态

**错误示例：**
```
File has not been read yet. Read it first before writing to it.
```

**正确流程：**
```
1. Read tool → 获取文件内容和时间戳
2. Write tool → 验证时间戳后写入
```

### 4.4 最佳实践

**路径规范：**
- 使用绝对路径，避免相对路径歧义
- 使用 `/` 分隔符（跨平台兼容）
- 避免UNC路径（`\\server\share`）的安全风险

**内容规范：**
- 保持一致的换行符格式
- 避免在代码中写入emoji（除非用户要求）
- 不要自动创建文档文件（除非用户明确要求）

**权限意识：**
- 了解危险文件列表（`.gitconfig`, `.bashrc`等）
- 知道危险目录（`.git`, `.vscode`, `.claude`）
- 理解不同模式下的权限行为

---

## 5. 安全检查

### 5.1 路径验证

**路径安全检查流程（`utils/permissions/filesystem.ts`）：**

```typescript
// 第1219-1239行：拒绝规则检查
const pathsToCheck = precomputedPathsToCheck ?? getPathsForPermissionCheck(path)
for (const pathToCheck of pathsToCheck) {
  const denyRule = matchingRuleForInput(pathToCheck, toolPermissionContext, 'edit', 'deny')
  if (denyRule) {
    return {
      behavior: 'deny',
      message: `Permission to edit ${path} has been denied.`,
    }
  }
}
```

**getPathsForPermissionCheck返回：**
- 原始路径
- 符号链接解析后的真实路径

**UNC路径防御（第182-184行）：**
```typescript
// SECURITY: Skip filesystem operations for UNC paths to prevent NTLM credential leaks.
if (fullFilePath.startsWith('\\\\') || fullFilePath.startsWith('//')) {
  return { result: true }  // 让权限检查处理UNC路径
}
```

### 5.2 权限检查流程

**完整权限检查流程（`checkWritePermissionForTool`函数）：**

```
┌─────────────────────────────────────────────┐
│ 1. 检查拒绝规则（deny rules）                  │
│    → 匹配则返回 deny                         │
├─────────────────────────────────────────────┤
│ 2. 检查内部可编辑路径（plan files, scratchpad）│
│    → 匹配则允许                              │
├─────────────────────────────────────────────┤
│ 3. 检查.claude/**会话级允许规则               │
│    → 匹配特定格式则允许                       │
├─────────────────────────────────────────────┤
│ 4. 安全检查（Windows模式、危险文件/目录）      │
│    → 危险则返回 ask                          │
├─────────────────────────────────────────────┤
│ 5. 检查询问规则（ask rules）                  │
│    → 匹配则返回 ask                          │
├─────────────────────────────────────────────┤
│ 6. 检查工作目录（acceptEdits模式）            │
│    → 在工作目录内则允许                       │
├─────────────────────────────────────────────┤
│ 7. 检查允许规则（allow rules）                │
│    → 匹配则允许                              │
├─────────────────────────────────────────────┤
│ 8. 默认返回 ask                              │
└─────────────────────────────────────────────┘
```

### 5.3 覆盖确认机制

**危险文件列表（第57-68行）：**
```typescript
export const DANGEROUS_FILES = [
  '.gitconfig',
  '.gitmodules',
  '.bashrc',
  '.bash_profile',
  '.zshrc',
  '.zprofile',
  '.profile',
  '.ripgreprc',
  '.mcp.json',
  '.claude.json',
] as const
```

**危险目录列表（第74-79行）：**
```typescript
export const DANGEROUS_DIRECTORIES = [
  '.git',
  '.vscode',
  '.idea',
  '.claude',
] as const
```

**安全检查函数（`checkPathSafetyForAutoEdit`）：**
- 检查路径是否匹配危险文件/目录
- 检查Windows可疑路径模式
- 检查Claude配置文件保护

---

## 6. 从零实现

### 6.1 核心工具结构

```typescript
// tools/FileWriteTool/FileWriteTool.ts

import { z } from 'zod/v4'
import { buildTool, type ToolDef } from '../../Tool.js'

// 输入Schema
const inputSchema = lazySchema(() =>
  z.strictObject({
    file_path: z.string().describe('绝对路径'),
    content: z.string().describe('文件内容'),
  }),
)

// 输出Schema
const outputSchema = lazySchema(() =>
  z.object({
    type: z.enum(['create', 'update']),
    filePath: z.string(),
    content: z.string(),
    structuredPatch: z.array(hunkSchema()),
    originalFile: z.string().nullable(),
    gitDiff: gitDiffSchema().optional(),
  }),
)

// 工具定义
export const FileWriteTool = buildTool({
  name: 'Write',
  searchHint: 'create or overwrite files',
  maxResultSizeChars: 100_000,
  strict: true,
  
  async description() {
    return 'Write a file to the local filesystem.'
  },
  
  userFacingName,
  getToolUseSummary,
  
  // 核心方法
  async call({ file_path, content }, context) {
    // 1. 路径展开
    const fullFilePath = expandPath(file_path)
    
    // 2. 目录准备
    await fs.mkdir(dirname(fullFilePath))
    
    // 3. 历史备份（如果启用）
    if (fileHistoryEnabled()) {
      await fileHistoryTrackEdit(...)
    }
    
    // 4. 原子验证+写入
    let meta = readFileSyncWithMetadata(fullFilePath)
    if (meta !== null) {
      // 检查并发冲突
      const lastWriteTime = getFileModificationTime(fullFilePath)
      const lastRead = context.readFileState.get(fullFilePath)
      if (lastWriteTime > lastRead.timestamp) {
        throw new Error('File unexpectedly modified')
      }
    }
    
    // 5. 写入文件
    writeTextContent(fullFilePath, content, 'utf8', 'LF')
    
    // 6. 更新状态
    context.readFileState.set(fullFilePath, {
      content,
      timestamp: getFileModificationTime(fullFilePath),
    })
    
    // 7. 返回结果
    return { data: { type: meta ? 'update' : 'create', ... } }
  },
})
```

### 6.2 权限检查实现

```typescript
// utils/permissions/filesystem.ts

export function checkWritePermissionForTool<Input>(
  tool: Tool<Input>,
  input: Input,
  toolPermissionContext: ToolPermissionContext,
): PermissionDecision {
  const path = tool.getPath(input)
  const pathsToCheck = getPathsForPermissionCheck(path)
  
  // 1. 拒绝规则
  for (const pathToCheck of pathsToCheck) {
    if (matchingRuleForInput(pathToCheck, context, 'edit', 'deny')) {
      return { behavior: 'deny', message: 'Permission denied' }
    }
  }
  
  // 2. 安全检查
  const safetyCheck = checkPathSafetyForAutoEdit(path, pathsToCheck)
  if (!safetyCheck.safe) {
    return { behavior: 'ask', message: safetyCheck.message }
  }
  
  // 3. 允许规则
  if (matchingRuleForInput(path, context, 'edit', 'allow')) {
    return { behavior: 'allow' }
  }
  
  // 4. 默认询问
  return { behavior: 'ask', message: 'Permission required' }
}
```

### 6.3 输入验证实现

```typescript
// validateInput 方法实现要点

async validateInput({ file_path, content }, context) {
  const fullFilePath = expandPath(file_path)
  
  // 1. Secret检查
  const secretError = checkTeamMemSecrets(fullFilePath, content)
  if (secretError) return { result: false, message: secretError }
  
  // 2. 拒绝规则
  if (matchingRuleForInput(fullFilePath, context, 'edit', 'deny')) {
    return { result: false, message: 'Denied by permission settings' }
  }
  
  // 3. UNC路径安全检查
  if (fullFilePath.startsWith('\\\\')) {
    return { result: true }  // 让权限系统处理
  }
  
  // 4. 文件存在性检查
  try {
    const stat = await fs.stat(fullFilePath)
    const readTimestamp = context.readFileState.get(fullFilePath)
    
    // 5. 必须先读取
    if (!readTimestamp) {
      return { result: false, message: 'File has not been read yet' }
    }
    
    // 6. 并发修改检查
    if (stat.mtimeMs > readTimestamp.timestamp) {
      return { result: false, message: 'File modified since read' }
    }
  } catch (e) {
    if (isENOENT(e)) {
      return { result: true }  // 新文件允许写入
    }
    throw e
  }
  
  return { result: true }
}
```

### 6.4 UI渲染实现

```typescript
// tools/FileWriteTool/UI.tsx

const MAX_LINES_TO_RENDER = 10

// 创建文件消息
function FileWriteToolCreatedMessage({ filePath, content, verbose }) {
  const numLines = countLines(content)
  const plusLines = numLines - MAX_LINES_TO_RENDER
  
  return (
    <Box flexDirection="column">
      <Text>Wrote <Text bold>{numLines}</Text> lines to <Text bold>{filePath}</Text></Text>
      <HighlightedCode 
        code={verbose ? content : content.split("\n").slice(0, MAX_LINES_TO_RENDER).join("\n")}
        filePath={filePath}
      />
      {!verbose && plusLines > 0 && (
        <Text dimColor>… +{plusLines} lines <CtrlOToExpand /></Text>
      )}
    </Box>
  )
}

// 结果截断判断
export function isResultTruncated({ type, content }: Output): boolean {
  if (type !== 'create') return false  // update显示完整diff
  
  // 检查是否有超过MAX_LINES_TO_RENDER行
  let pos = 0
  for (let i = 0; i < MAX_LINES_TO_RENDER; i++) {
    pos = content.indexOf('\n', pos)
    if (pos === -1) return false
    pos++
  }
  return pos < content.length
}
```

---

## 7. 输出Schema与结果映射

### 7.1 输出Schema定义

```typescript
const outputSchema = lazySchema(() =>
  z.object({
    type: z.enum(['create', 'update']).describe('创建新文件或更新已有文件'),
    filePath: z.string().describe('写入的文件路径'),
    content: z.string().describe('写入的内容'),
    structuredPatch: z.array(hunkSchema()).describe('差异patch'),
    originalFile: z.string().nullable().describe('原始内容（新文件为null）'),
    gitDiff: gitDiffSchema().optional(),
  }),
)
```

### 7.2 结果映射到ToolResultBlock

```typescript
// 第418-433行
mapToolResultToToolResultBlockParam({ filePath, type }, toolUseID) {
  switch (type) {
    case 'create':
      return {
        tool_use_id: toolUseID,
        type: 'tool_result',
        content: `File created successfully at: ${filePath}`,
      }
    case 'update':
      return {
        tool_use_id: toolUseID,
        type: 'tool_result',
        content: `The file ${filePath} has been updated successfully.`,
      }
  }
}
```

---

## 8. 辅助系统集成

### 8.1 LSP服务器通知

写入完成后通知LSP服务器（第308-326行）：

```typescript
const lspManager = getLspServerManager()
if (lspManager) {
  // 清除已传递的诊断
  clearDeliveredDiagnosticsForFile(`file://${fullFilePath}`)
  
  // 通知文件变更
  lspManager.changeFile(fullFilePath, content).catch(...)
  
  // 通知文件保存（触发TypeScript服务器诊断）
  lspManager.saveFile(fullFilePath).catch(...)
}
```

### 8.2 VSCode通知

```typescript
// 第329行
notifyVscodeFileUpdated(fullFilePath, oldContent, content)
```

用于VSCode扩展的diff视图更新。

### 8.3 Skills激活

```typescript
// 第232-245行
const newSkillDirs = await discoverSkillDirsForPaths([fullFilePath], cwd)
if (newSkillDirs.length > 0) {
  dynamicSkillDirTriggers?.add(dir)
  addSkillDirectories(newSkillDirs).catch(() => {})
}

activateConditionalSkillsForPaths([fullFilePath], cwd)
```

如果写入的文件路径触发skill目录发现，会自动激活相关skills。

### 8.4 文件历史追踪

```typescript
// 第255-264行
if (fileHistoryEnabled()) {
  await fileHistoryTrackEdit(
    updateFileHistoryState,
    fullFilePath,
    parentMessage.uuid,
  )
}
```

启用文件历史时，备份原始内容用于undo功能。

---

## 9. 错误处理

### 9.1 验证错误码

| errorCode | 错误类型 | 用户提示 |
|-----------|----------|----------|
| 0 | Team memory secrets | 检测到敏感信息 |
| 1 | 权限拒绝 | 目录被权限设置拒绝 |
| 2 | 未读取文件 | 文件尚未读取，请先Read |
| 3 | 并发修改 | 文件读取后被修改，请重新Read |

### 9.2 运行时错误

**FILE_UNEXPECTEDLY_MODIFIED_ERROR：**
```typescript
// tools/FileEditTool/constants.ts 第10-11行
export const FILE_UNEXPECTEDLY_MODIFIED_ERROR =
  'File has been unexpectedly modified. Read it again before attempting to write it.'
```

当时间戳检查失败且内容比对不一致时抛出。

---

## 10. 总结

Write工具是Claude Code CLI的文件写入核心，具有以下关键特性：

1. **强制先读后写**：防止覆盖未知内容，支持并发冲突检测
2. **原子性操作**：读取-验证-写入在临界区连续执行
3. **多层权限检查**：拒绝规则→安全检查→允许规则→默认询问
4. **路径安全防护**：UNC路径防御、危险文件/目录保护
5. **系统集成**：LSP通知、VSCode同步、Skills激活、文件历史

用户使用时应注意：
- 已存在文件必须先Read
- 使用绝对路径
- 了解Write vs Edit的选择时机
- 遵循权限系统的交互流程

---

*文档生成：2026-04-02*
*源码位置：`tools/FileWriteTool/`*