# NotebookEditTool 工具详解

**分析日期:** 2026-04-02

---

## 1. 概述

### 1.1 工具作用

NotebookEditTool 是 Claude Code CLI 中专门用于编辑 Jupyter Notebook (.ipynb) 文件的工具。它提供了完整的单元格(Cell)级别编辑能力，包括替换、插入和删除操作。

**核心文件位置:**
- 主实现: `tools/NotebookEditTool/NotebookEditTool.ts`
- 常量定义: `tools/NotebookEditTool/constants.ts`
- 提示词: `tools/NotebookEditTool/prompt.ts`
- UI渲染: `tools/NotebookEditTool/UI.tsx`

### 1.2 Jupyter Notebook 支持

Jupyter Notebook 是一种交互式文档格式，广泛用于:
- 数据分析和可视化
- 机器学习实验
- 科学计算
- 教学和文档编写

Notebook 文件 (.ipynb) 本质上是 JSON 文件，包含:
- 元数据 (metadata): 语言信息、内核信息等
- 单元格列表 (cells): 代码单元格或 Markdown 单元格
- 格式版本 (nbformat): Notebook 格式版本号

---

## 2. 输入 Schema

### 2.1 完整参数定义

```typescript
// 位置: tools/NotebookEditTool/NotebookEditTool.ts:30-57
z.strictObject({
  notebook_path: z.string().describe(
    'The absolute path to the Jupyter notebook file to edit (must be absolute, not relative)'
  ),
  cell_id: z.string().optional().describe(
    'The ID of the cell to edit. When inserting a new cell, the new cell will be inserted after the cell with this ID, or at the beginning if not specified.'
  ),
  new_source: z.string().describe('The new source for the cell'),
  cell_type: z.enum(['code', 'markdown']).optional().describe(
    'The type of the cell (code or markdown). If not specified, it defaults to the current cell type. If using edit_mode=insert, this is required.'
  ),
  edit_mode: z.enum(['replace', 'insert', 'delete']).optional().describe(
    'The type of edit to make (replace, insert, delete). Defaults to replace.'
  )
})
```

### 2.2 参数详解

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `notebook_path` | string | 是 | Notebook 文件的绝对路径 |
| `cell_id` | string | 否 | 要操作的单元格 ID，支持两种格式 |
| `new_source` | string | 是 | 单元格的新内容 |
| `cell_type` | 'code' \| 'markdown' | 条件 | 单元格类型，insert 模式下必填 |
| `edit_mode` | 'replace' \| 'insert' \| 'delete' | 否 | 编辑模式，默认为 replace |

### 2.3 cell_id 支持的格式

`cell_id` 参数支持两种格式:

1. **真实 ID 格式**: Jupyter Notebook 的单元格真实 ID (nbformat 4.5+)
   - 例如: `"abc123def456"`

2. **索引格式**: `cell-N` 形式的数字索引
   - 例如: `"cell-0"`, `"cell-1"`, `"cell-2"`

```typescript
// 位置: utils/notebook.ts:217-224
export function parseCellId(cellId: string): number | undefined {
  const match = cellId.match(/^cell-(\d+)$/)
  if (match && match[1]) {
    const index = parseInt(match[1], 10)
    return isNaN(index) ? undefined : index
  }
  return undefined
}
```

---

## 3. 编辑模式

### 3.1 replace 模式 (替换)

**用途:** 替换现有单元格的内容

**行为:**
- 查找指定 `cell_id` 对应的单元格
- 替换该单元格的 `source` 内容
- 如果是代码单元格，自动重置 `execution_count` 为 `null` 并清空 `outputs`
- 可选地通过 `cell_type` 改变单元格类型

**示例:**
```json
{
  "notebook_path": "/path/to/notebook.ipynb",
  "cell_id": "cell-0",
  "new_source": "print('Hello, World!')",
  "edit_mode": "replace"
}
```

**特殊处理:**
```typescript
// 位置: tools/NotebookEditTool/NotebookEditTool.ts:370-377
// 如果 cellIndex 等于单元格总数，自动转换为 insert 模式
let edit_mode = originalEditMode
if (edit_mode === 'replace' && cellIndex === notebook.cells.length) {
  edit_mode = 'insert'
  if (!cell_type) {
    cell_type = 'code' // 默认为代码单元格
  }
}
```

### 3.2 insert 模式 (插入)

**用途:** 在指定位置插入新单元格

**行为:**
- 如果指定了 `cell_id`，在该单元格之后插入
- 如果未指定 `cell_id`，在开头插入 (index 0)
- `cell_type` 参数必填

**单元格 ID 生成:**
```typescript
// 位置: tools/NotebookEditTool/NotebookEditTool.ts:380-390
// 对于 nbformat > 4 或 (nbformat === 4 且 nbformat_minor >= 5)
// 生成随机 ID
if (notebook.nbformat > 4 ||
    (notebook.nbformat === 4 && notebook.nbformat_minor >= 5)) {
  if (edit_mode === 'insert') {
    new_cell_id = Math.random().toString(36).substring(2, 15)
  }
}
```

**新单元格结构:**

代码单元格:
```typescript
// 位置: tools/NotebookEditTool/NotebookEditTool.ts:404-412
{
  cell_type: 'code',
  id: new_cell_id,
  source: new_source,
  metadata: {},
  execution_count: null,
  outputs: []
}
```

Markdown 单元格:
```typescript
// 位置: tools/NotebookEditTool/NotebookEditTool.ts:397-403
{
  cell_type: 'markdown',
  id: new_cell_id,
  source: new_source,
  metadata: {}
}
```

**示例:**
```json
{
  "notebook_path": "/path/to/notebook.ipynb",
  "cell_id": "cell-0",
  "new_source": "# 新章节\n这是新的 Markdown 内容",
  "cell_type": "markdown",
  "edit_mode": "insert"
}
```

### 3.3 delete 模式 (删除)

**用途:** 删除指定单元格

**行为:**
- 查找指定 `cell_id` 对应的单元格
- 从 notebook.cells 数组中移除该单元格

**实现:**
```typescript
// 位置: tools/NotebookEditTool/NotebookEditTool.ts:392-394
if (edit_mode === 'delete') {
  notebook.cells.splice(cellIndex, 1)
}
```

**示例:**
```json
{
  "notebook_path": "/path/to/notebook.ipynb",
  "cell_id": "cell-2",
  "new_source": "",  // 删除模式下仍需提供，但被忽略
  "edit_mode": "delete"
}
```

---

## 4. 用户使用指南

### 4.1 前置条件：读取文件

**重要:** NotebookEditTool 强制要求先读取文件才能编辑。这是为了防止:
- 编辑从未查看过的文件
- 基于过时视图进行编辑导致数据丢失

```typescript
// 位置: tools/NotebookEditTool/NotebookEditTool.ts:218-237
// 验证文件是否已读取
const readTimestamp = toolUseContext.readFileState.get(fullPath)
if (!readTimestamp) {
  return {
    result: false,
    message: 'File has not been read yet. Read it first before writing to it.',
    errorCode: 9,
  }
}

// 验证文件是否在读取后被修改
if (getFileModificationTime(fullPath) > readTimestamp.timestamp) {
  return {
    result: false,
    message: 'File has been modified since read, either by the user or by a linter. Read it again before attempting to write it.',
    errorCode: 10,
  }
}
```

### 4.2 使用 Claude Code CLI 编辑 Notebook

**步骤 1: 读取 Notebook**

首先让 Claude 读取目标 notebook 文件:

```
请读取 /path/to/notebook.ipynb 文件
```

**步骤 2: 查看单元格信息**

Claude 会显示所有单元格及其 ID:
```
<cell id="cell-0"># 第一个单元格内容</cell>
<cell id="cell-1">print('代码示例')</cell>
```

**步骤 3: 执行编辑**

根据需要请求编辑操作:

```
请将 cell-0 的内容替换为 "# 标题已更新"
```

```
请在 cell-1 后插入一个新的 markdown 单元格，内容为 "## 新章节"
```

```
请删除 cell-2 这个单元格
```

### 4.3 最佳实践

#### 4.3.1 单元格操作顺序

1. **总是先读取** - 确保了解当前 notebook 状态
2. **使用正确的 cell_id** - 确认目标单元格的 ID
3. **明确指定 cell_type** - 插入新单元格时必须指定类型

#### 4.3.2 文件路径规范

- **必须使用绝对路径** - 相对路径不被支持
- UNC 路径 (Windows 网络路径) 会被跳过验证以防止 NTLM 凭据泄露

```typescript
// 位置: tools/NotebookEditTool/NotebookEditTool.ts:184-187
// 安全性: 跳过 UNC 路径的文件系统操作，防止 NTLM 凭据泄露
if (fullPath.startsWith('\\\\') || fullPath.startsWith('//')) {
  return { result: true }
}
```

#### 4.3.3 错误处理

工具会返回详细的错误信息:

| errorCode | 说明 |
|-----------|------|
| 1 | Notebook 文件不存在 |
| 2 | 文件不是 .ipynb 格式 |
| 4 | edit_mode 值无效 |
| 5 | insert 模式未指定 cell_type |
| 6 | Notebook JSON 格式无效 |
| 7 | 指定索引的单元格不存在 |
| 8 | 指定 ID 的单元格未找到 |
| 9 | 文件尚未读取 |
| 10 | 文件在读取后被修改 |

### 4.4 输出 Schema

工具执行成功后返回的数据结构:

```typescript
// 位置: tools/NotebookEditTool/NotebookEditTool.ts:60-85
{
  new_source: string,        // 写入单元格的新内容
  cell_id: string,           // 操作的单元格 ID
  cell_type: 'code' | 'markdown', // 单元格类型
  language: string,          // Notebook 的编程语言
  edit_mode: string,         // 使用的编辑模式
  error: string | undefined, // 错误信息（如果有）
  notebook_path: string,     // Notebook 文件路径
  original_file: string,     // 修改前的文件内容
  updated_file: string       // 修改后的文件内容
}
```

---

## 5. 从零实现

### 5.1 工具定义结构

使用 `buildTool` 函数创建工具:

```typescript
// 位置: tools/NotebookEditTool/NotebookEditTool.ts:90-490
export const NotebookEditTool = buildTool({
  name: NOTEBOOK_EDIT_TOOL_NAME,  // 'NotebookEdit'
  searchHint: 'edit Jupyter notebook cells (.ipynb)',
  maxResultSizeChars: 100_000,
  shouldDefer: true,
  async description() {
    return DESCRIPTION  // 'Replace the contents of a specific cell in a Jupyter notebook.'
  },
  async prompt() {
    return PROMPT  // 详细的工具使用说明
  },
  userFacingName() {
    return 'Edit Notebook'
  },
  // ... 其他配置
})
```

### 5.2 核心方法实现

#### 5.2.1 权限检查

```typescript
// 位置: tools/NotebookEditTool/NotebookEditTool.ts:125-132
async checkPermissions(input, context): Promise<PermissionDecision> {
  const appState = context.getAppState()
  return checkWritePermissionForTool(
    NotebookEditTool,
    input,
    appState.toolPermissionContext,
  )
}
```

#### 5.2.2 输入验证

```typescript
// 位置: tools/NotebookEditTool/NotebookEditTool.ts:176-294
async validateInput({ notebook_path, cell_type, cell_id, edit_mode = 'replace' }, toolUseContext) {
  // 1. 解析绝对路径
  const fullPath = isAbsolute(notebook_path)
    ? notebook_path
    : resolve(getCwd(), notebook_path)

  // 2. 跳过 UNC 路径安全检查
  if (fullPath.startsWith('\\\\') || fullPath.startsWith('//')) {
    return { result: true }
  }

  // 3. 验证文件扩展名
  if (extname(fullPath) !== '.ipynb') {
    return {
      result: false,
      message: 'File must be a Jupyter notebook (.ipynb file)...',
      errorCode: 2,
    }
  }

  // 4. 验证编辑模式
  if (!['replace', 'insert', 'delete'].includes(edit_mode)) {
    return {
      result: false,
      message: 'Edit mode must be replace, insert, or delete.',
      errorCode: 4,
    }
  }

  // 5. 验证 insert 模式必须指定 cell_type
  if (edit_mode === 'insert' && !cell_type) {
    return {
      result: false,
      message: 'Cell type is required when using edit_mode=insert.',
      errorCode: 5,
    }
  }

  // 6. 读取前置检查
  const readTimestamp = toolUseContext.readFileState.get(fullPath)
  if (!readTimestamp) {
    return { result: false, message: 'File has not been read yet...', errorCode: 9 }
  }

  // 7. 文件修改时间检查
  if (getFileModificationTime(fullPath) > readTimestamp.timestamp) {
    return { result: false, message: 'File has been modified since read...', errorCode: 10 }
  }

  // 8. 验证文件存在和格式有效
  // 9. 验证 cell_id 存在
  // ...
}
```

#### 5.2.3 核心执行逻辑

```typescript
// 位置: tools/NotebookEditTool/NotebookEditTool.ts:295-489
async call({ notebook_path, new_source, cell_id, cell_type, edit_mode }, context) {
  const fullPath = isAbsolute(notebook_path)
    ? notebook_path
    : resolve(getCwd(), notebook_path)

  // 1. 文件历史追踪
  if (fileHistoryEnabled()) {
    await fileHistoryTrackEdit(updateFileHistoryState, fullPath, parentMessage.uuid)
  }

  try {
    // 2. 读取文件内容和元数据
    const { content, encoding, lineEndings } = readFileSyncWithMetadata(fullPath)

    // 3. 解析 JSON (使用非缓存版本以避免缓存污染)
    let notebook = jsonParse(content) as NotebookContent

    // 4. 定位单元格索引
    let cellIndex
    if (!cell_id) {
      cellIndex = 0  // 默认在开头插入
    } else {
      cellIndex = notebook.cells.findIndex(cell => cell.id === cell_id)
      if (cellIndex === -1) {
        const parsedCellIndex = parseCellId(cell_id)
        if (parsedCellIndex !== undefined) {
          cellIndex = parsedCellIndex
        }
      }
      if (edit_mode === 'insert') {
        cellIndex += 1  // 在指定单元格之后插入
      }
    }

    // 5. 执行编辑操作
    if (edit_mode === 'delete') {
      notebook.cells.splice(cellIndex, 1)
    } else if (edit_mode === 'insert') {
      const new_cell = cell_type === 'markdown'
        ? { cell_type: 'markdown', id: new_cell_id, source: new_source, metadata: {} }
        : { cell_type: 'code', id: new_cell_id, source: new_source, metadata: {},
            execution_count: null, outputs: [] }
      notebook.cells.splice(cellIndex, 0, new_cell)
    } else {
      // replace 模式
      const targetCell = notebook.cells[cellIndex]
      targetCell.source = new_source
      if (targetCell.cell_type === 'code') {
        targetCell.execution_count = null
        targetCell.outputs = []
      }
    }

    // 6. 写回文件
    const updatedContent = jsonStringify(notebook, null, 1)  // 使用缩进 1
    writeTextContent(fullPath, updatedContent, encoding, lineEndings)

    // 7. 更新文件状态缓存
    readFileState.set(fullPath, {
      content: updatedContent,
      timestamp: getFileModificationTime(fullPath),
      offset: undefined,
      limit: undefined,
    })

    return { data: { /* 成功数据 */ } }
  } catch (error) {
    return { data: { /* 错误数据 */ } }
  }
}
```

### 5.3 UI 渲染组件

UI.tsx 提供了多个渲染函数:

```typescript
// 位置: tools/NotebookEditTool/UI.tsx

// 工具使用摘要
export function getToolUseSummary(input): string | null

// 工具使用消息渲染
export function renderToolUseMessage(input, { verbose }): React.ReactNode

// 工具拒绝消息渲染
export function renderToolUseRejectedMessage(input, { verbose }): React.ReactNode

// 工具错误消息渲染
export function renderToolUseErrorMessage(result, { verbose }): React.ReactNode

// 工具结果消息渲染
export function renderToolResultMessage(output): React.ReactNode
```

### 5.4 结果映射

将工具结果映射为 API 响应格式:

```typescript
// 位置: tools/NotebookEditTool/NotebookEditTool.ts:133-171
mapToolResultToToolResultBlockParam({ cell_id, edit_mode, new_source, error }, toolUseID) {
  if (error) {
    return {
      tool_use_id: toolUseID,
      type: 'tool_result',
      content: error,
      is_error: true,
    }
  }

  switch (edit_mode) {
    case 'replace':
      return {
        tool_use_id: toolUseID,
        type: 'tool_result',
        content: `Updated cell ${cell_id} with ${new_source}`,
      }
    case 'insert':
      return {
        tool_use_id: toolUseID,
        type: 'tool_result',
        content: `Inserted cell ${cell_id} with ${new_source}`,
      }
    case 'delete':
      return {
        tool_use_id: toolUseID,
        type: 'tool_result',
        content: `Deleted cell ${cell_id}`,
      }
    default:
      return {
        tool_use_id: toolUseID,
        type: 'tool_result',
        content: 'Unknown edit mode',
      }
  }
}
```

### 5.5 完整实现示例

以下是一个简化版的 NotebookEditTool 实现框架:

```typescript
import { z } from 'zod/v4'
import { buildTool } from '../../Tool.js'
import { readFileSyncWithMetadata, writeTextContent } from '../../utils/file.js'
import { jsonParse, jsonStringify } from '../../utils/slowOperations.js'

// 输入 Schema
const inputSchema = z.strictObject({
  notebook_path: z.string(),
  cell_id: z.string().optional(),
  new_source: z.string(),
  cell_type: z.enum(['code', 'markdown']).optional(),
  edit_mode: z.enum(['replace', 'insert', 'delete']).optional(),
})

// 输出 Schema
const outputSchema = z.object({
  cell_id: z.string().optional(),
  cell_type: z.enum(['code', 'markdown']),
  new_source: z.string(),
  edit_mode: z.string(),
  language: z.string(),
  error: z.string().optional(),
})

// 构建工具
export const NotebookEditTool = buildTool({
  name: 'NotebookEdit',
  searchHint: 'edit Jupyter notebook cells (.ipynb)',

  async validateInput(input, context) {
    // 验证逻辑
    return { result: true }
  },

  async call(input, context) {
    const notebook = jsonParse(content)

    switch (input.edit_mode) {
      case 'delete':
        notebook.cells.splice(cellIndex, 1)
        break
      case 'insert':
        notebook.cells.splice(cellIndex, 0, newCell)
        break
      default: // replace
        notebook.cells[cellIndex].source = input.new_source
    }

    writeTextContent(path, jsonStringify(notebook, null, 1))
    return { data: { /* 输出数据 */ } }
  },
})
```

---

## 6. 辅助工具函数

### 6.1 单元格 ID 解析

```typescript
// 位置: utils/notebook.ts:217-224
export function parseCellId(cellId: string): number | undefined {
  const match = cellId.match(/^cell-(\d+)$/)
  if (match && match[1]) {
    const index = parseInt(match[1], 10)
    return isNaN(index) ? undefined : index
  }
  return undefined
}
```

### 6.2 Notebook 读取

```typescript
// 位置: utils/notebook.ts:164-183
export async function readNotebook(
  notebookPath: string,
  cellId?: string,
): Promise<NotebookCellSource[]> {
  const fullPath = expandPath(notebookPath)
  const buffer = await getFsImplementation().readFileBytes(fullPath)
  const content = buffer.toString('utf-8')
  const notebook = jsonParse(content) as NotebookContent
  const language = notebook.metadata.language_info?.name ?? 'python'

  if (cellId) {
    const cell = notebook.cells.find(c => c.id === cellId)
    if (!cell) {
      throw new Error(`Cell with ID "${cellId}" not found in notebook`)
    }
    return [processCell(cell, notebook.cells.indexOf(cell), language, true)]
  }
  return notebook.cells.map((cell, index) =>
    processCell(cell, index, language, false),
  )
}
```

### 6.3 单元格处理

```typescript
// 位置: utils/notebook.ts:83-117
function processCell(
  cell: NotebookCell,
  index: number,
  codeLanguage: string,
  includeLargeOutputs: boolean,
): NotebookCellSource {
  const cellId = cell.id ?? `cell-${index}`
  const cellData: NotebookCellSource = {
    cellType: cell.cell_type,
    source: Array.isArray(cell.source) ? cell.source.join('') : cell.source,
    execution_count: cell.cell_type === 'code' ? cell.execution_count || undefined : undefined,
    cell_id: cellId,
  }

  if (cell.cell_type === 'code') {
    cellData.language = codeLanguage
  }

  if (cell.cell_type === 'code' && cell.outputs?.length) {
    const outputs = cell.outputs.map(processOutput)
    if (!includeLargeOutputs && isLargeOutputs(outputs)) {
      cellData.outputs = [{ output_type: 'stream', text: 'Outputs too large...' }]
    } else {
      cellData.outputs = outputs
    }
  }

  return cellData
}
```

---

## 7. 关键设计决策

### 7.1 为什么强制先读取?

```typescript
// 位置: tools/NotebookEditTool/NotebookEditTool.ts:218-220
// 没有 "读取-后编辑" 约束，模型可能编辑从未见过的文件，
// 或基于过时视图进行编辑 —— 造成静默数据丢失
```

这防止了两个关键问题:
1. **盲目编辑**: 编辑从未查看过的文件
2. **过时视图**: 文件在读取后被外部修改，导致覆盖他人更改

### 7.2 JSON 解析为何不用缓存版本?

```typescript
// 位置: tools/NotebookEditTool/NotebookEditTool.ts:327-332
// 必须使用非缓存的 jsonParse:
// safeParseJSON 按内容字符串缓存并返回共享对象引用，
// 但我们在下面会就地修改 notebook (cells.splice, targetCell.source = ...)。
// 使用缓存版本会污染 validateInput() 和后续 call() 的缓存。
```

### 7.3 为什么 offset 设为 undefined?

```typescript
// 位置: tools/NotebookEditTool/NotebookEditTool.ts:434-440
// offset:undefined 会破坏 FileReadTool 的去重匹配 —— 没有这个，
// 同一毫秒内的 Read→NotebookEdit→Read 会返回针对过时上下文内容的
// file_unchanged 存根。
readFileState.set(fullPath, {
  content: updatedContent,
  timestamp: getFileModificationTime(fullPath),
  offset: undefined,  // 关键：破坏去重匹配
  limit: undefined,
})
```

---

## 8. 总结

NotebookEditTool 是一个设计完善的 Jupyter Notebook 编辑工具，具有以下特点:

1. **安全的编辑机制**: 强制先读取后编辑，防止数据丢失
2. **灵活的操作模式**: 支持替换、插入、删除三种模式
3. **完善的错误处理**: 详细的错误码和错误信息
4. **智能的 ID 解析**: 支持真实 ID 和索引格式
5. **格式保持**: 保持原文件的编码和换行符风格
6. **版本兼容**: 正确处理不同 nbformat 版本的 ID 生成

使用该工具时，务必遵循"先读取后编辑"的原则，并使用绝对路径指定文件位置。