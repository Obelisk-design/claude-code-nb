# Rewind回退机制详解

**分析日期:** 2026-04-02

## 1. Rewind概述

### 1.1 核心概念

Rewind（回退）是Claude Code CLI提供的时间旅行功能，允许用户将对话和代码状态恢复到历史任意节点。该机制实现双层回退：
- **对话回退**：截断消息历史到指定节点，重置相关状态
- **代码回退**：通过文件快照系统恢复磁盘文件到历史版本

### 1.2 入口命令

```typescript
// commands/rewind/index.ts
const rewind = {
  description: `Restore the code and/or conversation to a previous point`,
  name: 'rewind',
  aliases: ['checkpoint'],  // 别名checkpoint
  argumentHint: '',
  type: 'local',            // 本地命令，不触发API请求
  supportsNonInteractive: false,
  load: () => import('./rewind.js'),
}
```

**触发方式：**
- 命令行输入 `/rewind` 或 `/checkpoint`
- 快捷键 `Ctrl+E`（可通过keybindings配置）
- 自动触发：中断操作时自动检测并跳过确认

### 1.3 架构位置

```
commands/rewind/
├── index.ts          # 命令注册入口
└── rewind.ts         # 命令实现（触发MessageSelector）

components/MessageSelector.tsx  # 回退UI组件（核心）
screens/REPL.tsx                 # 回退逻辑实现
utils/fileHistory.ts             # 文件历史快照系统
```

---

## 2. commands/rewind/ 逐行分析

### 2.1 index.ts - 命令注册

```typescript
// 第1-13行：命令元数据定义
import type { Command } from '../../commands.js'

const rewind = {
  description: `Restore the code and/or conversation to a previous point`,
  name: 'rewind',
  aliases: ['checkpoint'],    // 支持两种命令名
  argumentHint: '',           // 不需要参数提示
  type: 'local',              // 本地命令类型
  supportsNonInteractive: false,  // 仅交互模式
  load: () => import('./rewind.js'),  // 懒加载实现
} satisfies Command

export default rewind
```

**设计要点：**
- `type: 'local'` 标识为本地命令，不消耗API token
- `aliases` 提供备用命令名，增强可发现性
- 懒加载减少启动时的模块加载开销

### 2.2 rewind.ts - 命令实现

```typescript
// 第1-13行：完整的命令执行逻辑
import type { LocalCommandResult } from '../../commands.js'
import type { ToolUseContext } from '../../Tool.js'

export async function call(
  _args: string,           // 接收但忽略命令参数
  context: ToolUseContext, // 工具使用上下文
): Promise<LocalCommandResult> {
  if (context.openMessageSelector) {
    context.openMessageSelector()  // 打开消息选择器
  }
  // Return a skip message to not append any messages.
  return { type: 'skip' }  // 不追加任何消息到历史
}
```

**关键流程：**
1. 检查 `openMessageSelector` 是否存在（仅在REPL交互模式存在）
2. 调用选择器打开回退界面
3. 返回 `{ type: 'skip' }` 表示命令本身不产生消息

---

## 3. 回退算法

### 3.1 rewindConversationTo - 对话截断算法

```typescript
// screens/REPL.tsx 第3661-3707行
const rewindConversationTo = useCallback((message: UserMessage) => {
  // Step 1: 获取当前消息列表（通过ref避免闭包陷阱）
  const prev = messagesRef.current;
  const messageIndex = prev.lastIndexOf(message);
  if (messageIndex === -1) return;  // 消息不存在则退出
  
  // Step 2: 记录回退事件（用于分析）
  logEvent('tengu_conversation_rewind', {
    preRewindMessageCount: prev.length,
    postRewindMessageCount: messageIndex,
    messagesRemoved: prev.length - messageIndex,
    rewindToMessageIndex: messageIndex
  });
  
  // Step 3: 截断消息数组
  setMessages(prev.slice(0, messageIndex));
  
  // Step 4: 重置会话ID（产生新会话分支）
  // 必须在setMessages之后执行
  setConversationId(randomUUID());
  
  // Step 5: 重置微压缩状态
  // 防止缓存的编辑引用已截断消息的tool_use_id
  resetMicrocompactState();
  
  // Step 6: 重置上下文折叠状态（可选功能）
  if (feature('CONTEXT_COLLAPSE')) {
    (require('../services/contextCollapse/index.js'))
      .resetContextCollapse();
  }
  
  // Step 7: 恢复权限模式和清除提示建议
  setAppState(prev => ({
    ...prev,
    // 从目标消息恢复权限模式
    toolPermissionContext: message.permissionMode && 
      prev.toolPermissionContext.mode !== message.permissionMode 
        ? { ...prev.toolPermissionContext, mode: message.permissionMode }
        : prev.toolPermissionContext,
    // 清除过期的提示建议
    promptSuggestion: {
      text: null,
      promptId: null,
      shownAt: 0,
      acceptedAt: 0,
      generationRequestId: null
    }
  }));
}, [setMessages, setAppState]);
```

**算法特点：**
- **O(n)查找**：使用lastIndexOf定位消息
- **O(1)截断**：slice操作高效
- **状态原子性**：所有状态更新在同一回调中完成
- **分支会话**：新UUID创建独立会话分支

### 3.2 restoreMessageSync - 同步回退算法

```typescript
// screens/REPL.tsx 第3712-3739行
const restoreMessageSync = useCallback((message: UserMessage) => {
  // Step 1: 执行对话截断
  rewindConversationTo(message);
  
  // Step 2: 提取消息文本用于重新提交
  const r = textForResubmit(message);
  if (r) {
    setInputValue(r.text);    // 填充输入框
    setInputMode(r.mode);     // 设置输入模式（bash/prompt）
  }
  
  // Step 3: 恢复粘贴的图片
  if (Array.isArray(message.message.content) && 
      message.message.content.some(block => block.type === 'image')) {
    const imageBlocks = message.message.content
      .filter(block => block.type === 'image');
    if (imageBlocks.length > 0) {
      const newPastedContents: Record<number, PastedContent> = {};
      imageBlocks.forEach((block, index) => {
        if (block.source.type === 'base64') {
          const id = message.imagePasteIds?.[index] ?? index + 1;
          newPastedContents[id] = {
            id,
            type: 'image',
            content: block.source.data,
            mediaType: block.source.media_type
          };
        }
      });
      setPastedContents(newPastedContents);
    }
  }
}, [rewindConversationTo, setInputValue]);
```

**设计要点：**
- **同步执行**：用于自动恢复场景，与abort的setMessages批处理合并
- **输入恢复**：提取原始输入文本填充到输入框
- **图片恢复**：保留粘贴的图片内容

### 3.3 textForResubmit - 输入文本提取

```typescript
// utils/messages.ts 第2873-2886行
export function textForResubmit(
  msg: UserMessage,
): { text: string; mode: 'bash' | 'prompt' } | null {
  // Step 1: 提取消息文本
  const content = getUserMessageText(msg)
  if (content === null) return null
  
  // Step 2: 检查bash-input标签（bash命令）
  const bash = extractTag(content, 'bash-input')
  if (bash) return { text: bash, mode: 'bash' }
  
  // Step 3: 检查command标签（斜杠命令）
  const cmd = extractTag(content, COMMAND_NAME_TAG)
  if (cmd) {
    const args = extractTag(content, COMMAND_ARGS_TAG) ?? ''
    return { text: `${cmd} ${args}`, mode: 'prompt' }
  }
  
  // Step 4: 普通文本，清除IDE上下文标签
  return { text: stripIdeContextTags(content), mode: 'prompt' }
}
```

---

## 4. 状态恢复

### 4.1 文件历史快照系统

#### 4.1.1 数据结构

```typescript
// utils/fileHistory.ts 第33-52行
export type FileHistoryBackup = {
  backupFileName: BackupFileName  // 备份文件名或null（表示文件不存在）
  version: number                 // 版本号，递增
  backupTime: Date                // 备份时间
}

export type FileHistorySnapshot = {
  messageId: UUID                 // 关联的消息ID
  trackedFileBackups: Record<string, FileHistoryBackup>  // 文件路径到备份的映射
  timestamp: Date                 // 快照时间
}

export type FileHistoryState = {
  snapshots: FileHistorySnapshot[]  // 快照数组（最多100个）
  trackedFiles: Set<string>         // 被跟踪的文件路径集合
  snapshotSequence: number          // 快照序列号（单调递增）
}

const MAX_SNAPSHOTS = 100  // 最大快照数量限制
```

#### 4.1.2 快照创建流程

```typescript
// utils/fileHistory.ts 第198-342行
export async function fileHistoryMakeSnapshot(
  updateFileHistoryState: (updater: (prev: FileHistoryState) => FileHistoryState) => void,
  messageId: UUID,
): Promise<void> {
  if (!fileHistoryEnabled()) return undefined
  
  // Phase 1: 捕获当前状态（无操作更新器）
  let captured: FileHistoryState | undefined
  updateFileHistoryState(state => {
    captured = state
    return state  // 返回相同引用，避免额外渲染
  })
  if (!captured) return
  
  // Phase 2: 异步备份所有跟踪文件
  const trackedFileBackups: Record<string, FileHistoryBackup> = {}
  const mostRecentSnapshot = captured.snapshots.at(-1)
  if (mostRecentSnapshot) {
    await Promise.all(
      Array.from(captured.trackedFiles, async trackingPath => {
        try {
          const filePath = maybeExpandFilePath(trackingPath)
          const latestBackup = mostRecentSnapshot.trackedFileBackups[trackingPath]
          const nextVersion = latestBackup ? latestBackup.version + 1 : 1
          
          // Stat检查文件是否存在
          let fileStats: Stats | undefined
          try {
            fileStats = await stat(filePath)
          } catch (e: unknown) {
            if (!isENOENT(e)) throw e
          }
          
          if (!fileStats) {
            // 文件被删除，记录null备份
            trackedFileBackups[trackingPath] = {
              backupFileName: null,
              version: nextVersion,
              backupTime: new Date(),
            }
            return
          }
          
          // 检查是否需要新备份（mtime优化）
          if (latestBackup && latestBackup.backupFileName !== null &&
              !(await checkOriginFileChanged(filePath, latestBackup.backupFileName, fileStats))) {
            // 文件未修改，复用旧备份
            trackedFileBackups[trackingPath] = latestBackup
            return
          }
          
          // 创建新备份
          trackedFileBackups[trackingPath] = await createBackup(filePath, nextVersion)
        } catch (error) {
          logError(error)
        }
      }),
    )
  }
  
  // Phase 3: 提交新快照到状态
  updateFileHistoryState((state: FileHistoryState) => {
    // 继承phase 2期间trackEdit添加的文件
    const lastSnapshot = state.snapshots.at(-1)
    if (lastSnapshot) {
      for (const trackingPath of state.trackedFiles) {
        if (trackingPath in trackedFileBackups) continue
        const inherited = lastSnapshot.trackedFileBackups[trackingPath]
        if (inherited) trackedFileBackups[trackingPath] = inherited
      }
    }
    
    const newSnapshot: FileHistorySnapshot = {
      messageId,
      trackedFileBackups,
      timestamp: new Date(),
    }
    
    const allSnapshots = [...state.snapshots, newSnapshot]
    const updatedState: FileHistoryState = {
      ...state,
      snapshots: allSnapshots.length > MAX_SNAPSHOTS 
        ? allSnapshots.slice(-MAX_SNAPSHOTS)  // 保留最新100个
        : allSnapshots,
      snapshotSequence: (state.snapshotSequence ?? 0) + 1,
    }
    
    // 记录到会话存储（用于resume支持）
    void recordFileHistorySnapshot(messageId, newSnapshot, false)
    
    return updatedState
  })
}
```

#### 4.1.3 文件回退算法

```typescript
// utils/fileHistory.ts 第347-397行
export async function fileHistoryRewind(
  updateFileHistoryState: (updater: (prev: FileHistoryState) => FileHistoryState) => void,
  messageId: UUID,
): Promise<void> {
  if (!fileHistoryEnabled()) return
  
  // 捕获状态（无操作更新器）
  let captured: FileHistoryState | undefined
  updateFileHistoryState(state => {
    captured = state
    return state
  })
  if (!captured) return
  
  // 查找目标快照
  const targetSnapshot = captured.snapshots.findLast(
    snapshot => snapshot.messageId === messageId,
  )
  if (!targetSnapshot) {
    throw new Error('The selected snapshot was not found')
  }
  
  // 应用快照到文件系统
  const filesChanged = await applySnapshot(captured, targetSnapshot)
  
  logEvent('tengu_file_history_rewind_success', {
    trackedFilesCount: captured.trackedFiles.size,
    filesChangedCount: filesChanged.length,
  })
}
```

#### 4.1.4 applySnapshot - 文件恢复核心

```typescript
// utils/fileHistory.ts 第537-591行
async function applySnapshot(
  state: FileHistoryState,
  targetSnapshot: FileHistorySnapshot,
): Promise<string[]> {
  const filesChanged: string[] = []
  
  for (const trackingPath of state.trackedFiles) {
    try {
      const filePath = maybeExpandFilePath(trackingPath)
      const targetBackup = targetSnapshot.trackedFileBackups[trackingPath]
      
      // 确定备份文件名
      const backupFileName: BackupFileName | undefined = targetBackup
        ? targetBackup.backupFileName
        : getBackupFileNameFirstVersion(trackingPath, state)
      
      if (backupFileName === undefined) {
        // 无法解析备份，跳过
        continue
      }
      
      if (backupFileName === null) {
        // 目标版本文件不存在，删除当前文件
        try {
          await unlink(filePath)
          filesChanged.push(filePath)
        } catch (e: unknown) {
          if (!isENOENT(e)) throw e
          // 文件已不存在，无需操作
        }
        continue
      }
      
      // 文件应存在于特定版本，仅在差异时恢复
      if (await checkOriginFileChanged(filePath, backupFileName)) {
        await restoreBackup(filePath, backupFileName)
        filesChanged.push(filePath)
      }
    } catch (error) {
      logError(error)
    }
  }
  return filesChanged
}
```

### 4.2 变更检测算法

```typescript
// utils/fileHistory.ts 第600-672行
export async function checkOriginFileChanged(
  originalFile: string,
  backupFileName: string,
  originalStatsHint?: Stats,
): Promise<boolean> {
  const backupPath = resolveBackupPath(backupFileName)
  
  // Stat两个文件
  let originalStats: Stats | null = originalStatsHint ?? null
  if (!originalStats) {
    try {
      originalStats = await stat(originalFile)
    } catch (e: unknown) {
      if (!isENOENT(e)) return true
    }
  }
  
  let backupStats: Stats | null = null
  try {
    backupStats = await stat(backupPath)
  } catch (e: unknown) {
    if (!isENOENT(e)) return true
  }
  
  return compareStatsAndContent(originalStats, backupStats, async () => {
    try {
      const [originalContent, backupContent] = await Promise.all([
        readFile(originalFile, 'utf-8'),
        readFile(backupPath, 'utf-8'),
      ])
      return originalContent !== backupContent
    } catch {
      return true  // 读取失败视为已变更
    }
  })
}

function compareStatsAndContent<T extends boolean | Promise<boolean>>(
  originalStats: Stats | null,
  backupStats: Stats | null,
  compareContent: () => T,
): T | boolean {
  // 一个存在一个不存在 -> 已变更
  if ((originalStats === null) !== (backupStats === null)) {
    return true
  }
  // 都不存在 -> 未变更
  if (originalStats === null || backupStats === null) {
    return false
  }
  
  // 检查mode和size
  if (originalStats.mode !== backupStats.mode || 
      originalStats.size !== backupStats.size) {
    return true
  }
  
  // mtime优化：如果原文件mtime早于备份，无需内容比较
  if (originalStats.mtimeMs < backupStats.mtimeMs) {
    return false
  }
  
  // 执行内容比较
  return compareContent()
}
```

### 4.3 Diff统计计算

```typescript
// utils/fileHistory.ts 第414-484行
export async function fileHistoryGetDiffStats(
  state: FileHistoryState,
  messageId: UUID,
): Promise<DiffStats> {
  if (!fileHistoryEnabled()) return undefined
  
  const targetSnapshot = state.snapshots.findLast(
    snapshot => snapshot.messageId === messageId,
  )
  if (!targetSnapshot) return undefined
  
  const results = await Promise.all(
    Array.from(state.trackedFiles, async trackingPath => {
      try {
        const filePath = maybeExpandFilePath(trackingPath)
        const targetBackup = targetSnapshot.trackedFileBackups[trackingPath]
        const backupFileName = targetBackup
          ? targetBackup.backupFileName
          : getBackupFileNameFirstVersion(trackingPath, state)
        
        if (backupFileName === undefined) return null
        
        const stats = await computeDiffStatsForFile(
          filePath,
          backupFileName === null ? undefined : backupFileName,
        )
        if (stats?.insertions || stats?.deletions) {
          return { filePath, stats }
        }
        // 处理零字节文件的特殊情况
        if (backupFileName === null && (await pathExists(filePath))) {
          return { filePath, stats }
        }
        return null
      } catch (error) {
        return null
      }
    }),
  )
  
  // 汇总统计
  const filesChanged: string[] = []
  let insertions = 0
  let deletions = 0
  for (const r of results) {
    if (!r) continue
    filesChanged.push(r.filePath)
    insertions += r.stats?.insertions || 0
    deletions += r.stats?.deletions || 0
  }
  return { filesChanged, insertions, deletions }
}
```

---

## 5. 用户使用指南

### 5.1 快捷键绑定

```typescript
// keybindings/defaultBindings.ts 第246-265行
{
  context: 'MessageSelector',
  bindings: {
    up: 'messageSelector:up',
    down: 'messageSelector:down',
    k: 'messageSelector:up',      // Vim风格
    j: 'messageSelector:down',    // Vim风格
    'ctrl+p': 'messageSelector:up',
    'ctrl+n': 'messageSelector:down',
    'ctrl+up': 'messageSelector:top',
    'shift+up': 'messageSelector:top',
    'meta+up': 'messageSelector:top',
    'shift+k': 'messageSelector:top',
    'ctrl+down': 'messageSelector:bottom',
    'shift+down': 'messageSelector:bottom',
    'meta+down': 'messageSelector:bottom',
    'shift+j': 'messageSelector:bottom',
    enter: 'messageSelector:select',
  },
}
```

**操作指南：**

| 按键 | 功能 |
|------|------|
| `↑/k/Ctrl+p` | 向上移动选择 |
| `↓/j/Ctrl+n` | 向下移动选择 |
| `Ctrl+↑/Shift+↑/Shift+k` | 跳转到顶部 |
| `Ctrl+↓/Shift+↓/Shift+j` | 跳转到底部 |
| `Enter` | 选择当前消息 |
| `Esc` | 关闭选择器/返回列表 |

### 5.2 回退选项

```typescript
// components/MessageSelector.tsx 第93-134行
function getRestoreOptions(canRestoreCode: boolean): OptionWithDescription<RestoreOption>[] {
  const baseOptions: OptionWithDescription<RestoreOption>[] = canRestoreCode ? [
    { value: 'both', label: 'Restore code and conversation' },
    { value: 'conversation', label: 'Restore conversation' },
    { value: 'code', label: 'Restore code' },
  ] : [
    { value: 'conversation', label: 'Restore conversation' },
  ];
  
  // 添加总结选项（带可选输入）
  baseOptions.push({
    value: 'summarize',
    label: 'Summarize from here',
    type: 'input',
    placeholder: 'add context (optional)',
    ...
  });
  
  // 添加向上总结选项（仅ant版本）
  if ("external" === 'ant') {
    baseOptions.push({
      value: 'summarize_up_to',
      label: 'Summarize up to here',
      ...
    });
  }
  
  baseOptions.push({
    value: 'nevermind',
    label: 'Never mind',
  });
  
  return baseOptions;
}
```

**选项说明：**

| 选项 | 功能 | 适用场景 |
|------|------|----------|
| `both` | 同时恢复代码和对话 | 全面回退 |
| `conversation` | 仅恢复对话，代码不变 | 探索不同路径 |
| `code` | 仅恢复代码，对话保留 | 保持上下文但撤销修改 |
| `summarize` | 从选中点向后总结 | 保留早期历史，压缩近期 |
| `summarize_up_to` | 向前总结到选中点 | 保留近期历史，压缩早期 |
| `nevermind` | 取消操作 | 误触发时退出 |

### 5.3 消息过滤规则

```typescript
// components/MessageSelector.tsx 第767-792行
export function selectableUserMessagesFilter(message: Message): message is UserMessage {
  // 仅选择用户消息
  if (message.type !== 'user') {
    return false;
  }
  // 排除tool_result开头的消息
  if (Array.isArray(message.message.content) && 
      message.message.content[0]?.type === 'tool_result') {
    return false;
  }
  // 排除合成消息
  if (isSyntheticMessage(message)) {
    return false;
  }
  // 排除元消息（如系统提示）
  if (message.isMeta) {
    return false;
  }
  // 排除仅 transcripts 可见的消息
  if (message.isVisibleInTranscriptOnly) {
    return false;
  }
  return true;
}
```

### 5.4 自动跳过确认

```typescript
// screens/REPL.tsx 第3767-3786行（messageActionCaps.handleUuidSelection片段）
const rawIdx = findRawIndex(uuid);
const raw = rawIdx >= 0 ? messages[rawIdx] : undefined;
if (!raw || !selectableUserMessagesFilter(raw)) return;

// 检查是否可以自动恢复（无文件变更+仅有合成消息）
const noFileChanges = !(await fileHistoryHasAnyChanges(fileHistory, raw.uuid));
const onlySynthetic = messagesAfterAreOnlySynthetic(messages, rawIdx);

if (noFileChanges && onlySynthetic) {
  // 自动回退，跳过确认对话框
  onCancel();  // 先取消当前操作
  void handleRestoreMessage(raw);
} else {
  // 显示确认对话框
  setMessageSelectorPreselect(raw);
  setIsMessageSelectorVisible(true);
}
```

---

## 6. 从零实现

### 6.1 最小实现框架

```typescript
// Step 1: 定义核心类型
type BackupFileName = string | null;
type FileHistoryBackup = {
  backupFileName: BackupFileName;
  version: number;
  backupTime: Date;
};
type FileHistorySnapshot = {
  messageId: UUID;
  trackedFileBackups: Record<string, FileHistoryBackup>;
  timestamp: Date;
};
type FileHistoryState = {
  snapshots: FileHistorySnapshot[];
  trackedFiles: Set<string>;
};

// Step 2: 创建备份文件
async function createBackup(filePath: string, version: number): Promise<FileHistoryBackup> {
  const fileNameHash = createHash('sha256')
    .update(filePath)
    .digest('hex')
    .slice(0, 16);
  const backupFileName = `${fileNameHash}@v${version}`;
  const backupPath = join(configDir, 'file-history', sessionId, backupFileName);
  
  try {
    await copyFile(filePath, backupPath);
  } catch (e) {
    if (isENOENT(e)) {
      // 文件不存在，记录null备份
      return { backupFileName: null, version, backupTime: new Date() };
    }
    throw e;
  }
  
  return { backupFileName, version, backupTime: new Date() };
}

// Step 3: 恢复备份
async function restoreBackup(filePath: string, backupFileName: string): Promise<void> {
  const backupPath = join(configDir, 'file-history', sessionId, backupFileName);
  await copyFile(backupPath, filePath);
}

// Step 4: 检查文件变更
async function checkFileChanged(filePath: string, backupFileName: string): Promise<boolean> {
  const backupPath = join(configDir, 'file-history', sessionId, backupFileName);
  const [original, backup] = await Promise.all([
    readFile(filePath, 'utf-8').catch(() => null),
    readFile(backupPath, 'utf-8').catch(() => null),
  ]);
  return original !== backup;
}

// Step 5: 应用快照
async function applySnapshot(state: FileHistoryState, target: FileHistorySnapshot): Promise<string[]> {
  const changed: string[] = [];
  for (const path of state.trackedFiles) {
    const backup = target.trackedFileBackups[path];
    if (!backup) continue;
    
    if (backup.backupFileName === null) {
      // 目标版本文件不存在，删除当前
      await unlink(path).catch(() => {});
      changed.push(path);
    } else if (await checkFileChanged(path, backup.backupFileName)) {
      await restoreBackup(path, backup.backupFileName);
      changed.push(path);
    }
  }
  return changed;
}

// Step 6: 对话截断
function rewindConversation(messages: Message[], targetIndex: number): Message[] {
  return messages.slice(0, targetIndex);
}
```

### 6.2 状态管理集成

```typescript
// Step 1: 定义AppState扩展
type AppState = {
  fileHistory: FileHistoryState;
  // ...其他状态
};

// Step 2: 创建更新器
function updateFileHistory(
  setState: (f: (prev: AppState) => AppState) => void,
  updater: (prev: FileHistoryState) => FileHistoryState
): void {
  setState(prev => ({
    ...prev,
    fileHistory: updater(prev.fileHistory),
  }));
}

// Step 3: 跟踪文件编辑（在EditTool/WriteTool中调用）
async function trackFileEdit(filePath: string, messageId: UUID): Promise<void> {
  const backup = await createBackup(filePath, 1);
  updateFileHistory(setState, state => ({
    ...state,
    trackedFiles: new Set(state.trackedFiles).add(filePath),
    snapshots: [{
      messageId,
      trackedFileBackups: { [filePath]: backup },
      timestamp: new Date(),
    }],
  }));
}

// Step 4: 执行回退
async function executeRewind(
  message: UserMessage,
  messages: Message[],
  setState: (f: (prev: AppState) => AppState) => void,
  setMessages: (msgs: Message[]) => void,
): Promise<void> {
  // 截断对话
  const index = messages.lastIndexOf(message);
  if (index === -1) return;
  setMessages(messages.slice(0, index));
  
  // 恢复文件
  let state: FileHistoryState | undefined;
  setState(prev => {
    state = prev.fileHistory;
    return prev;
  });
  
  if (state) {
    const snapshot = state.snapshots.find(s => s.messageId === message.uuid);
    if (snapshot) {
      await applySnapshot(state, snapshot);
    }
  }
}
```

### 6.3 React组件实现

```typescript
// components/RewindSelector.tsx
import { useState, useCallback } from 'react';

type Props = {
  messages: Message[];
  onRewind: (message: UserMessage, restoreCode: boolean) => Promise<void>;
  onClose: () => void;
};

export function RewindSelector({ messages, onRewind, onClose }: Props) {
  const [selectedIndex, setSelectedIndex] = useState(0);
  const [isRestoring, setIsRestoring] = useState(false);
  
  // 过滤可选择的用户消息
  const selectableMessages = messages.filter(
    m => m.type === 'user' && !m.isMeta && !isSyntheticMessage(m)
  );
  
  // 键盘导航
  const handleKeyDown = useCallback((key: string) => {
    switch (key) {
      case 'up':
        setSelectedIndex(Math.max(0, selectedIndex - 1));
        break;
      case 'down':
        setSelectedIndex(Math.min(selectableMessages.length - 1, selectedIndex + 1));
        break;
      case 'enter':
        void handleSelect();
        break;
      case 'escape':
        onClose();
        break;
    }
  }, [selectedIndex, selectableMessages.length]);
  
  // 选择并执行回退
  const handleSelect = async () => {
    const message = selectableMessages[selectedIndex] as UserMessage;
    setIsRestoring(true);
    try {
      await onRewind(message, true);  // 默认同时恢复代码
      onClose();
    } catch (error) {
      console.error('Rewind failed:', error);
    }
    setIsRestoring(false);
  };
  
  return (
    <Box flexDirection="column">
      <Text bold>Rewind to:</Text>
      {selectableMessages.map((msg, idx) => (
        <Text key={msg.uuid} color={idx === selectedIndex ? 'cyan' : 'white'}>
          {idx === selectedIndex ? '> ' : '  '}
          {getMessagePreview(msg)}
        </Text>
      ))}
      <Text dimColor>Enter to select · Esc to cancel</Text>
    </Box>
  );
}
```

### 6.4 命令注册

```typescript
// commands/rewind.ts
import type { Command, LocalCommandResult } from '../commands.js';

export const rewindCommand: Command = {
  name: 'rewind',
  aliases: ['checkpoint'],
  description: 'Restore to a previous point',
  type: 'local',
  supportsNonInteractive: false,
  load: () => Promise.resolve({ call }),
};

export async function call(
  args: string,
  context: ToolUseContext,
): Promise<LocalCommandResult> {
  // 触发UI选择器
  if (context.openMessageSelector) {
    context.openMessageSelector();
  }
  return { type: 'skip' };
}
```

---

## 7. 关键文件路径索引

| 功能模块 | 文件路径 |
|----------|----------|
| 命令注册 | `commands/rewind/index.ts` |
| 命令实现 | `commands/rewind/rewind.ts` |
| 消息选择器UI | `components/MessageSelector.tsx` |
| 回退逻辑 | `screens/REPL.tsx` (第3661-3747行) |
| 文件历史系统 | `utils/fileHistory.ts` |
| 消息工具函数 | `utils/messages.ts` |
| 快捷键绑定 | `keybindings/defaultBindings.ts` |
| 快捷键schema | `keybindings/schema.ts` |
| 微压缩状态 | `services/compact/microCompact.ts` |
| 部分压缩 | `services/compact/compact.ts` |

---

*分析完成: 2026-04-02*