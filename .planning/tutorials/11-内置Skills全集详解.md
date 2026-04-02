# 11. Claude Code 内置 Skills 全集详解

**分析日期:** 2026-04-02

> **前置知识:** 阅读本文前，建议先阅读:
> - [09-扩展系统设计](./09-扩展系统设计.md) - 理解 Skills/Plugins/Hooks 三层扩展架构

---

## 1. 概述

### 1.1 Skills 系统设计理念

Claude Code CLI 的 Skills 系统是一个强大的扩展机制，允许用户通过 Markdown 文件定义可复用的工作流程和指令模板。Skills 系统的设计理念包括：

1. **声明式定义** - Skills 通过 Markdown 文件（`SKILL.md`）定义，包含 YAML frontmatter 和指令内容
2. **参数化模板** - 支持参数传递和动态替换，使 Skill 可适应不同场景
3. **权限控制** - 每个 Skill 可定义自己需要的工具权限，实现细粒度安全控制
4. **上下文隔离** - 支持 `inline`（当前会话）和 `fork`（独立子代理）两种执行模式
5. **动态发现** - 能够根据文件路径动态加载和激活相关 Skills

### 1.2 Bundled Skills vs 用户自定义 Skills

**Bundled Skills（内置 Skills）**:
- 编译进 CLI 二进制文件中，随 Claude Code 发行
- 所有用户默认可用（除非被 feature flag 限制）
- 通过 TypeScript 代码程序化注册
- 通常包含复杂的逻辑处理和动态内容生成
- 位置：`skills/bundled/` 目录

**用户自定义 Skills**:
- 以 Markdown 文件形式存在：`~/.claude/skills/` 或 `.claude/skills/`
- 用户可自由编辑和扩展
- 通过文件系统加载和解析
- 支持目录格式（`skill-name/SKILL.md`）
- 位置：用户配置目录或项目目录

**核心差异**:
| 特性 | Bundled Skills | 用户自定义 Skills |
|------|----------------|-------------------|
| 定义方式 | TypeScript 代码 | Markdown 文件 |
| 注册机制 | `registerBundledSkill()` | 文件扫描加载 |
| 动态逻辑 | 支持复杂 JS 逻辑 | 有限的参数替换 |
| 内容来源 | 程序生成 | 静态文件内容 |
| 可编辑性 | 不可编辑 | 可自由编辑 |

---

## 2. skills/bundledSkills.ts 分析

**文件位置**: `skills/bundledSkills.ts`

### 2.1 BundledSkillDefinition 类型定义

**位置**: bundledSkills.ts:15-41

```typescript
export type BundledSkillDefinition = {
  name: string                     // Skill 名称
  description: string              // 描述文本
  aliases?: string[]               // 可选别名
  whenToUse?: string               // 自动触发条件说明
  argumentHint?: string            // 参数提示文本
  allowedTools?: string[]          // 允许使用的工具列表
  model?: string                   // 使用的模型
  disableModelInvocation?: boolean // 是否禁用自动调用
  userInvocable?: boolean          // 用户是否可手动调用
  isEnabled?: () => boolean        // 动态启用条件
  hooks?: HooksSettings            // Hook 配置
  context?: 'inline' | 'fork'      // 执行上下文
  agent?: string                   // 使用的代理类型
  files?: Record<string, string>   // 附加文件内容
  getPromptForCommand: (           // Prompt 生成函数
    args: string,
    context: ToolUseContext,
  ) => Promise<ContentBlockParam[]>
}
```

### 2.2 registerBundledSkill() 函数实现

**位置**: bundledSkills.ts:53-100

**核心逻辑**:

1. **文件提取机制** (第 54-73 行):
   - 如果 Skill 定义包含 `files` 字段，会在首次调用时提取到磁盘
   - 使用闭包级记忆化（memoization）避免重复提取
   - 提取目录通过 `getBundledSkillExtractDir()` 生成

2. **Command 对象构建** (第 75-99 行):
   ```typescript
   const command: Command = {
     type: 'prompt',
     name: definition.name,
     description: definition.description,
     aliases: definition.aliases,
     hasUserSpecifiedDescription: true,
     allowedTools: definition.allowedTools ?? [],
     argumentHint: definition.argumentHint,
     whenToUse: definition.whenToUse,
     model: definition.model,
     disableModelInvocation: definition.disableModelInvocation ?? false,
     userInvocable: definition.userInvocable ?? true,
     contentLength: 0,
     source: 'bundled',
     loadedFrom: 'bundled',
     hooks: definition.hooks,
     skillRoot,
     context: definition.context,
     agent: definition.agent,
     isEnabled: definition.isEnabled,
     isHidden: !(definition.userInvocable ?? true),
     progressMessage: 'running',
     getPromptForCommand,
   }
   bundledSkills.push(command)
   ```

3. **安全文件写入** (第 186-193 行):
   - 使用 `O_NOFOLLOW` 和 `O_EXCL` 标志防止符号链接攻击
   - 使用 `0o600` 模式确保文件权限安全
   - Windows 平台使用字符串标志避免数值兼容问题

### 2.3 所有内置 Skills 注册列表

**位置**: bundledSkills.ts:24-79

**无条件注册的 Skills**（第 25-34 行）:
```typescript
registerUpdateConfigSkill()      // 配置更新
registerKeybindingsSkill()       // 键盘绑定帮助
registerVerifySkill()            // 验证（仅 ant 用户）
registerDebugSkill()             // 调试日志
registerLoremIpsumSkill()        // 填充文本生成（仅 ant）
registerSkillifySkill()          // Skill 创建助手（仅 ant）
registerRememberSkill()          // 内存管理
registerSimplifySkill()          // 代码简化
registerBatchSkill()             // 并行批处理
registerStuckSkill()             // 卡死诊断（仅 ant）
```

**条件注册的 Skills**（第 35-79 行）:
```typescript
// KAIROS 功能集
if (feature('KAIROS') || feature('KAIROS_DREAM')) {
  registerDreamSkill()
}

// 代码审查
if (feature('REVIEW_ARTIFACT')) {
  registerHunterSkill()
}

// 代理触发器
if (feature('AGENT_TRIGGERS')) {
  registerLoopSkill()  // 循环任务调度
}

// 远程代理调度
if (feature('AGENT_TRIGGERS_REMOTE')) {
  registerScheduleRemoteAgentsSkill()
}

// Claude API 开发
if (feature('BUILDING_CLAUDE_APPS')) {
  registerClaudeApiSkill()
}

// Chrome 浏览器自动化
if (shouldAutoEnableClaudeInChrome()) {
  registerClaudeInChromeSkill()
}

// Skill 生成器
if (feature('RUN_SKILL_GENERATOR')) {
  registerRunSkillGeneratorSkill()
}
```

---

## 3. 每个内置 Skill 逐行分析

### 3.1 debug.ts - 调试日志 Skill

**文件位置**: `skills/bundled/debug.ts`

#### Skill 基本信息

- **名称**: `debug`
- **描述**: Ant 用户看到完整事件日志描述，普通用户看到简化描述
- **工具权限**: `['Read', 'Grep', 'Glob']`
- **参数提示**: `[issue description]`
- **禁用自动调用**: `true`（用户需显式请求）

#### getPromptForCommand() 实现

**位置**: debug.ts:25-101

**核心逻辑**:

1. **启用调试日志** (第 28 行):
   ```typescript
   const wasAlreadyLogging = enableDebugLogging()
   ```
   调用 `enableDebugLogging()` 确保后续活动被记录。

2. **尾部读取优化** (第 34-49 行):
   ```typescript
   const stats = await stat(debugLogPath)
   const readSize = Math.min(stats.size, TAIL_READ_BYTES)  // 64KB
   const startOffset = stats.size - readSize
   ```
   - 只读取日志文件的最后 64KB
   - 避免长会话中读取整个文件导致内存峰值
   - 提取最后 20 行展示给用户

3. **Prompt 构建** (第 69-99 行):
   包含以下部分：
   - 调试 Skill 说明
   - 日志文件位置和大小信息
   - 最后 20 行日志内容
   - 用户问题描述
   - Settings 文件位置提示
   - 操作指令（查找错误、警告、栈追踪等）

#### 安全考虑

- 使用 `fs.stat()` 和 `fs.open()` 的低级 API 实现高效读取
- 提供 `isENOENT()` 检查处理文件不存在的情况
- 调用 `claudeCodeGuideAgent` 子代理提供额外帮助

---

### 3.2 claudeApi.ts - Claude API 开发 Skill

**文件位置**: `skills/bundled/claudeApi.ts`

#### Skill 基本信息

- **名称**: `claude-api`
- **描述**: 构建 Claude API 或 Anthropic SDK 应用
- **工具权限**: `['Read', 'Grep', 'Glob', 'WebFetch']`
- **触发条件**: 代码导入 `anthropic`/`@anthropic-ai/sdk`/`claude_agent_sdk`

#### 语言检测机制

**位置**: claudeApi.ts:30-53

```typescript
async function detectLanguage(): Promise<DetectedLanguage | null> {
  const cwd = getCwd()
  let entries: string[]
  try {
    entries = await readdir(cwd)  // 列出当前目录文件
  } catch {
    return null
  }

  // 根据文件扩展名和配置文件检测语言
  for (const [lang, indicators] of Object.entries(LANGUAGE_INDICATORS)) {
    for (const indicator of indicators) {
      if (indicator.startsWith('.')) {
        if (entries.some(e => e.endsWith(indicator))) return lang
      } else {
        if (entries.includes(indicator)) return lang
      }
    }
  }
  return null
}
```

**支持的语言**:
- Python: `.py`, `requirements.txt`, `pyproject.toml`, `setup.py`, `Pipfile`
- TypeScript: `.ts`, `.tsx`, `tsconfig.json`, `package.json`
- Java: `.java`, `pom.xml`, `build.gradle`
- Go: `.go`, `go.mod`
- Ruby: `.rb`, `Gemfile`
- C#: `.cs`, `.csproj`
- PHP: `.php`, `composer.json`

#### 内容处理机制

**位置**: claudeApi.ts:64-94

```typescript
function processContent(md: string, content: SkillContent): string {
  // 1. 剥离 HTML 注释
  let out = md
  do {
    prev = out
    out = out.replace(/<!--[\s\S]*?-->\n?/g, '')
  } while (out !== prev)

  // 2. 替换模型变量 {{VAR}}
  out = out.replace(/\{\{(\w+)\}\}/g, (match, key: string) =>
    (content.SKILL_MODEL_VARS as Record<string, string>)[key] ?? match
  )
  return out
}
```

#### Prompt 构建

**位置**: claudeApi.ts:132-178

```typescript
function buildPrompt(lang, args, content): string {
  // 1. 基础 Prompt（SKILL.md 内容直到 Reading Guide）
  const cleanPrompt = processContent(content.SKILL_PROMPT, content)
  const readingGuideIdx = cleanPrompt.indexOf('## Reading Guide')
  const basePrompt = readingGuideIdx !== -1
    ? cleanPrompt.slice(0, readingGuideIdx).trimEnd()
    : cleanPrompt

  // 2. 根据检测语言加载相关文档
  if (lang) {
    const filePaths = getFilesForLanguage(lang, content)
    parts.push(readingGuide)
    parts.push(buildInlineReference(filePaths, content))
  } else {
    // 无语言检测 - 包含所有文档
    parts.push('No project language was auto-detected...')
    parts.push(buildInlineReference(Object.keys(content.SKILL_FILES), content))
  }

  // 3. 保留 WebFetch 和 Common Pitfalls 部分
  const webFetchIdx = cleanPrompt.indexOf('## When to Use WebFetch')
  if (webFetchIdx !== -1) {
    parts.push(cleanPrompt.slice(webFetchIdx).trimEnd())
  }

  // 4. 用户请求
  if (args) {
    parts.push(`## User Request\n\n${args}`)
  }
}
```

#### 模型变量定义

**位置**: claudeApiContent.ts:36-45

```typescript
export const SKILL_MODEL_VARS = {
  OPUS_ID: 'claude-opus-4-6',
  OPUS_NAME: 'Claude Opus 4.6',
  SONNET_ID: 'claude-sonnet-4-6',
  SONNET_NAME: 'Claude Sonnet 4.6',
  HAIKU_ID: 'claude-haiku-4-5',
  HAIKU_NAME: 'Claude Haiku 4.5',
  PREV_SONNET_ID: 'claude-sonnet-4-5',
}
```

#### 内置文档结构

**位置**: claudeApiContent.ts:49-75

包含 247KB 的多语言文档：
- Python SDK 文档（README, streaming, tool-use, batches, files-api, agent-sdk）
- TypeScript SDK 文档（同上）
- 其他语言（C#, Go, Java, PHP, Ruby, curl）
- 共享文档（error-codes, models, prompt-caching, tool-use-concepts）

---

### 3.3 skillify.ts - Skill 创建助手

**文件位置**: `skills/bundled/skillify.ts`

#### Skill 基本信息

- **名称**: `skillify`
- **描述**: 捕获当前会话的可复用流程创建 Skill
- **工具权限**: `['Read', 'Write', 'Edit', 'Glob', 'Grep', 'AskUserQuestion', 'Bash(mkdir:*)']`
- **参数提示**: `[description of the process you want to capture]`
- **禁用自动调用**: `true`

#### 用户消息提取

**位置**: skillify.ts:6-20

```typescript
function extractUserMessages(messages: Message[]): string[] {
  return messages
    .filter((m): m is Extract<typeof m, { type: 'user' }> => m.type === 'user')
    .map(m => {
      const content = m.message.content
      if (typeof content === 'string') return content
      // 处理多块内容
      return content
        .filter((b): b is Extract<typeof b, { type: 'text' }> => b.type === 'text')
        .map(b => b.text)
        .join('\n')
    })
    .filter(text => text.trim().length > 0)
}
```

#### Prompt 模板结构

**位置**: skillify.ts:22-156

**SKILLIFY_PROMPT 包含以下部分**:

1. **会话上下文** (第 29-37 行):
   ```markdown
   ## Your Session Context
   <session_memory>
   {{sessionMemory}}
   </session_memory>
   <user_messages>
   {{userMessages}}
   </user_messages>
   ```

2. **Step 1: 分析会话** (第 40-51 行):
   - 识别可复用流程
   - 输入/参数分析
   - 步骤分解
   - 成功标准定义
   - 用户纠正点

3. **Step 2: 用户访谈** (第 53-88 行):
   - **Round 1**: 高级确认（名称、描述、目标）
   - **Round 2**: 详细信息（步骤、参数、保存位置）
   - **Round 3**: 步骤分解（产物、确认点、并行可能性）
   - **Round 4**: 最终问题（触发时机、注意事项）

4. **Step 3: 编写 SKILL.md** (第 90-128 行):
   提供 SKILL.md 格式模板：
   ```markdown
   ---
   name: {{skill-name}}
   description: {{one-line description}}
   allowed-tools:
     {{list of tool permission patterns}}
   when_to_use: {{trigger conditions}}
   argument-hint: "{{hint}}"
   arguments:
     {{list of argument names}}
   context: {{inline or fork}}
   ---
   ```

5. **Per-step 注解** (第 131-146 行):
   - **Success criteria**: 必需字段
   - **Execution**: `Direct`/`Task agent`/`Teammate`/`[human]`
   - **Artifacts**: 步骤产物
   - **Human checkpoint**: 需要用户确认的点
   - **Rules**: 硬性规则

#### getPromptForCommand 实现

**位置**: skillify.ts:179-196

```typescript
async getPromptForCommand(args, context) {
  const sessionMemory = (await getSessionMemoryContent()) ?? 'No session memory available.'
  const userMessages = extractUserMessages(
    getMessagesAfterCompactBoundary(context.messages)
  )

  const userDescriptionBlock = args
    ? `The user described this process as: "${args}"`
    : ''

  const prompt = SKILLIFY_PROMPT
    .replace('{{sessionMemory}}', sessionMemory)
    .replace('{{userMessages}}', userMessages.join('\n\n---\n\n'))
    .replace('{{userDescriptionBlock}}', userDescriptionBlock)

  return [{ type: 'text', text: prompt }]
}
```

---

### 3.4 verify.ts - 验证 Skill

**文件位置**: `skills/bundled/verify.ts`

#### Skill 基本信息

- **名称**: `verify`
- **描述**: 验证代码变更是否按预期工作
- **用户类型限制**: 仅 `ant` 用户可用（第 13-15 行检查）

#### 注册逻辑

**位置**: verify.ts:12-30

```typescript
export function registerVerifySkill(): void {
  if (process.env.USER_TYPE !== 'ant') {
    return  // 非 ant 用户直接跳过注册
  }

  registerBundledSkill({
    name: 'verify',
    description: DESCRIPTION,  // 从 frontmatter 提取
    userInvocable: true,
    files: SKILL_FILES,        // 附加示例文件
    async getPromptForCommand(args) {
      const parts: string[] = [SKILL_BODY.trimStart()]
      if (args) {
        parts.push(`## User Request\n\n${args}`)
      }
      return [{ type: 'text', text: parts.join('\n\n') }]
    },
  })
}
```

#### 内容加载机制

**位置**: verifyContent.ts

```typescript
import cliMd from './verify/examples/cli.md'
import serverMd from './verify/examples/server.md'
import skillMd from './verify/SKILL.md'

export const SKILL_MD: string = skillMd
export const SKILL_FILES: Record<string, string> = {
  'examples/cli.md': cliMd,
  'examples/server.md': serverMd,
}
```

通过 Bun 的 text loader 将 Markdown 文件内联为字符串。

---

### 3.5 batch.ts - 并行批处理 Skill

**文件位置**: `skills/bundled/batch.ts`

#### Skill 基本信息

- **名称**: `batch`
- **描述**: 研究和规划大规模变更，在 5-30 个隔离 worktree 代理中并行执行
- **参数提示**: `<instruction>`
- **禁用自动调用**: `true`

#### 核心常量

**位置**: batch.ts:9-18

```typescript
const MIN_AGENTS = 5
const MAX_AGENTS = 30

const WORKER_INSTRUCTIONS = `
After you finish implementing the change:
1. **Simplify** — Invoke the simplify skill
2. **Run unit tests** — Run project's test suite
3. **Test end-to-end** — Follow e2e test recipe
4. **Commit and push** — Create PR with gh pr create
5. **Report** — End with: PR: <url> or PR: none — <reason>
`
```

#### Prompt 构建逻辑

**位置**: batch.ts:19-89

**Phase 1: Research and Plan** (第 28-58 行):
1. 调用 `ENTER_PLAN_MODE_TOOL_NAME` 进入计划模式
2. 启动研究代理理解范围
3. 分解为 5-30 个独立单元：
   - 可独立实现
   - 可独立合并
   - 大致均匀大小
4. 确定端到端测试方法
5. 写入计划文件

**Phase 2: Spawn Workers** (第 60-75 行):
```typescript
// 所有代理必须使用 worktree 隔离和后台运行
spawn one background agent per work unit using AGENT_TOOL_NAME
All agents must use:
  isolation: "worktree"
  run_in_background: true
```

**Phase 3: Track Progress** (第 77-88 行):
- 渲染状态表格
- 解析代理结果中的 `PR: <url>` 行
- 更新表格状态和链接

#### Git 仓库检查

**位置**: batch.ts:110-123

```typescript
async getPromptForCommand(args) {
  const instruction = args.trim()
  if (!instruction) {
    return [{ type: 'text', text: MISSING_INSTRUCTION_MESSAGE }]
  }

  const isGit = await getIsGit()
  if (!isGit) {
    return [{ type: 'text', text: NOT_A_GIT_REPO_MESSAGE }]
  }

  return [{ type: 'text', text: buildPrompt(instruction) }]
}
```

必须要求 Git 仓库才能执行批处理操作。

---

### 3.6 loop.ts - 循环任务调度 Skill

**文件位置**: `skills/bundled/loop.ts`

#### Skill 基本信息

- **名称**: `loop`
- **描述**: 按间隔循环运行提示或斜杠命令
- **参数提示**: `[interval] <prompt>`
- **启用条件**: `isKairosCronEnabled()` 函数判断

#### 间隔解析规则

**位置**: loop.ts:25-44

```typescript
// 解析优先级：
// 1. 前导 token 匹配 ^\d+[smhd]$ → interval, rest = prompt
// 2. 结尾 "every <N><unit>" → 提取 interval, 剥离 prompt
// 3. 默认 interval = 10m, full input = prompt

// 示例：
// - "5m /babysit-prs" → interval 5m, prompt /babysit-prs (rule 1)
// - "check the deploy every 20m" → interval 20m, prompt check the deploy (rule 2)
// - "check the deploy" → interval 10m, prompt check the deploy (rule 3)
// - "check every PR" → interval 10m, prompt check every PR (rule 3)
```

#### Cron 表转换

**位置**: loop.ts:46-59

```typescript
// Interval → Cron 转换表：
// | Interval pattern      | Cron expression     |
// |-----------------------|---------------------|
// | Nm where N ≤ 59       | */N * * * *         |
// | Nm where N ≥ 60       | 0 */H * * *         |
// | Nh where N ≤ 23       | 0 */N * * *         |
// | Nd                    | 0 0 */N * *         |
// | Ns                    | ceil(N/60)m         |

// 间隔必须整除单位，否则选择最接近的干净间隔
```

#### 执行逻辑

**位置**: loop.ts:62-67

```typescript
// 1. 调用 CRON_CREATE_TOOL_NAME
// 2. 确认调度：cron 表达式、人类可读间隔、过期时间
// 3. 立即执行解析的 prompt（不等第一次 cron 触发）
```

---

### 3.7 remember.ts - 内存管理 Skill

**文件位置**: `skills/bundled/remember.ts`

#### Skill 基本信息

- **名称**: `remember`
- **描述**: 检查 auto-memory 条目并提议提升到 CLAUDE.md/CLAUDE.local.md
- **用户类型限制**: 仅 `ant` 用户
- **启用条件**: `isAutoMemoryEnabled()` 函数判断

#### 分类逻辑

**位置**: remember.ts:21-37

```typescript
// 分类表：
// | Destination        | What belongs there               |
// |--------------------|----------------------------------|
// | CLAUDE.md          | 项目级约定和指令                 |
// | CLAUDE.local.md    | 个人指令                         |
// | Team memory        | 跨仓库知识                       |
// | Stay in auto-mem   | 工作笔记、临时上下文             |
```

#### 工作流程

**位置**: remember.ts:15-61

1. **Gather all memory layers** (第 17-20 行): 读取所有内存层内容
2. **Classify each entry** (第 22-36 行): 分类每个条目
3. **Identify cleanup opportunities** (第 38-45 行): 识别清理机会（重复、过时、冲突）
4. **Present the report** (第 47-55 行): 展示报告（Promotions、Cleanup、Ambiguous、No action）

---

### 3.8 simplify.ts - 代码简化 Skill

**文件位置**: `skills/bundled/simplify.ts`

#### Skill 基本信息

- **名称**: `simplify`
- **描述**: 检查代码变更的复用性、质量和效率，修复发现的问题

#### 三阶段审查

**位置**: simplify.ts:12-53

**Phase 1: Identify Changes** (第 14-16 行):
```typescript
Run `git diff` to see what changed.
If no git changes, review most recently modified files.
```

**Phase 2: Launch Three Review Agents** (第 18-46 行):

1. **Agent 1: Code Reuse Review**:
   - 搜索现有工具和辅助函数
   - 标记重复功能
   - 标记可使用现有工具的代码

2. **Agent 2: Code Quality Review**:
   - 冗余状态
   - 参数泛滥
   - 复制粘贴变种
   - 泄漏抽象
   - 字符串化代码
   - 不必要 JSX 嵌套
   - 不必要注释

3. **Agent 3: Efficiency Review**:
   - 不必要工作
   - 错失并发机会
   - 热路径膨胀
   - 循环无操作更新
   - 不必要存在性检查
   - 内存问题
   - 过宽操作

**Phase 3: Fix Issues** (第 48-52 行):
等待所有代理完成，聚合发现并修复问题。

---

### 3.9 stuck.ts - 卡死诊断 Skill

**文件位置**: `skills/bundled/stuck.ts`

#### Skill 基本信息

- **名称**: `stuck`
- **描述**: 调查冻结/卡死/缓慢的 Claude Code 会话并发布报告到 Slack
- **用户类型限制**: 仅 `ant` 用户

#### 诊断指标

**位置**: stuck.ts:12-22

```typescript
// 卡死会话标志：
// - High CPU (≥90%) sustained → 无限循环
// - Process state 'D' → I/O 挂起
// - Process state 'T' → 用户 Ctrl+Z
// - Process state 'Z' → 父进程未回收
// - Very high RSS (≥4GB) → 内存泄漏
// - Stuck child process → 子进程挂起
```

#### 诊断步骤

**位置**: stuck.ts:24-39

```typescript
// 1. 列出所有 Claude Code 进程
ps -axo pid=,pcpu=,rss=,etime=,state=,comm=,command= | grep -E '(claude|cli)'

// 2. 检查可疑进程
pgrep -lP <pid>  // 子进程
ps -p <child_pid> -o command=  // 完整命令

// 3. 可选：栈转储
sample <pid> 3  // macOS
```

#### Slack 报告格式

**位置**: stuck.ts:41-54

```typescript
// 两消息结构：
// 1. Top-level message: 主机名、版本、简洁症状
// 2. Thread reply: 完整诊断转储（PID、CPU%、RSS、状态、诊断）
```

---

### 3.10 updateConfig.ts - 配置更新 Skill

**文件位置**: `skills/bundled/updateConfig.ts`

#### Skill 基本信息

- **名称**: `update-config`
- **描述**: 通过 settings.json 配置 Claude Code
- **工具权限**: `['Read']`

#### 配置文件位置

**位置**: updateConfig.ts:15-26

```typescript
// | File                      | Scope   | Git       |
// |---------------------------|---------|-----------|
// | ~/.claude/settings.json   | Global  | N/A       |
// | .claude/settings.json     | Project | Commit    |
// | .claude/settings.local.json | Project | Gitignore |
```

#### JSON Schema 生成

**位置**: updateConfig.ts:10-13

```typescript
function generateSettingsSchema(): string {
  const jsonSchema = toJSONSchema(SettingsSchema(), { io: 'input' })
  return jsonStringify(jsonSchema, null, 2)
}
```

动态生成 Schema 保持与类型同步。

#### Hooks 文档

**位置**: updateConfig.ts:110-267

包含完整的 Hooks 配置文档：
- Hook 事件类型
- Hook 类型（command、prompt、agent）
- 输入/输出格式
- 常见模式示例

#### Hook 验证流程

**位置**: updateConfig.ts:269-305

```typescript
// 6 步验证流程：
// 1. Dedup check - 读取目标文件检查现有 Hook
// 2. Construct command - 构建适合本项目的命令
// 3. Pipe-test - 测试命令是否工作
// 4. Write JSON - 合并到目标文件
// 5. Validate - 用 jq 验证语法
// 6. Prove - 触发匹配工具证明 Hook 触发
```

---

### 3.11 keybindings.ts - 键盘绑定帮助 Skill

**文件位置**: `skills/bundled/keybindings.ts`

#### Skill 基本信息

- **名称**: `keybindings-help`
- **描述**: 自定义键盘快捷键
- **工具权限**: `['Read']`
- **启用条件**: `isKeybindingCustomizationEnabled()`

#### 动态表格生成

**位置**: keybindings.ts:20-56

```typescript
function generateContextsTable(): string {
  return markdownTable(
    ['Context', 'Description'],
    KEYBINDING_CONTEXTS.map(ctx => [
      `\`${ctx}\``,
      KEYBINDING_CONTEXT_DESCRIPTIONS[ctx],
    ]),
  )
}

function generateActionsTable(): string {
  // 构建 action → { keys, context } 映射
  const actionInfo: Record<string, { keys: string[]; context: string }> = {}
  for (const block of DEFAULT_BINDINGS) {
    for (const [key, action] of Object.entries(block.bindings)) {
      if (action) {
        if (!actionInfo[action]) {
          actionInfo[action] = { keys: [], context: block.context }
        }
        actionInfo[action].keys.push(key)
      }
    }
  }
  // 生成表格...
}
```

#### Reserved Shortcuts

**位置**: keybindings.ts:89-112

```typescript
// 三类保留快捷键：
// 1. Non-rebindable (errors) - 不可重新绑定
// 2. Terminal reserved (errors/warnings) - 终端保留
// 3. macOS reserved (errors) - macOS 系统保留
```

#### 文件格式示例

**位置**: keybindings.ts:114-147

```typescript
const FILE_FORMAT_EXAMPLE = {
  $schema: 'https://www.schemastore.org/claude-code-keybindings.json',
  $docs: 'https://code.claude.com/docs/en/keybindings',
  bindings: [
    {
      context: 'Chat',
      bindings: {
        'ctrl+e': 'chat:externalEditor',
      },
    },
  ],
}
```

---

### 3.12 loremIpsum.ts - 填充文本 Skill

**文件位置**: `skills/bundled/loremIpsum.ts`

#### Skill 基本信息

- **名称**: `lorem-ipsum`
- **描述**: 生成长上下文测试的填充文本
- **用户类型限制**: 仅 `ant` 用户
- **参数提示**: `[token_count]`

#### 单 Token 单词列表

**位置**: loremIpsum.ts:5-200

```typescript
// 经过 API token 计数验证的单 token 英文单词
const ONE_TOKEN_WORDS = [
  // Articles & pronouns: the, a, an, I, you, ...
  // Common verbs: is, are, was, were, be, been, ...
  // Common nouns & adjectives: time, year, day, way, ...
  // Prepositions & conjunctions: in, on, at, to, for, ...
  // Common adverbs: not, now, just, more, also, ...
  // Tech/common words: test, code, data, file, line, ...
]
```

#### 生成逻辑

**位置**: loremIpsum.ts:202-232

```typescript
function generateLoremIpsum(targetTokens: number): string {
  let tokens = 0
  let result = ''

  while (tokens < targetTokens) {
    // Sentence: 10-20 words
    const sentenceLength = 10 + Math.floor(Math.random() * 11)
    // ... 生成句子 ...

    // Paragraph break every 5-8 sentences (20% chance)
    if (wordsInSentence > 0 && Math.random() < 0.2 && tokens < targetTokens) {
      result += '\n\n'
    }
  }

  return result.trim()
}
```

#### 安全上限

**位置**: loremIpsum.ts:259-262

```typescript
// Cap at 500k tokens for safety
const cappedTokens = Math.min(targetTokens, 500_000)
```

---

### 3.13 claudeInChrome.ts - Chrome 自动化 Skill

**文件位置**: `skills/bundled/claudeInChrome.ts`

#### Skill 基本信息

- **名称**: `claude-in-chrome`
- **描述**: 自动化 Chrome 浏览器交互
- **启用条件**: `shouldAutoEnableClaudeInChrome()`
- **触发时机**: 用户想与网页交互、截图、读取控制台日志时

#### MCP 工具映射

**位置**: claudeInChrome.ts:6-9

```typescript
const CLAUDE_IN_CHROME_MCP_TOOLS = BROWSER_TOOLS.map(
  tool => `mcp__claude-in-chrome__${tool.name}`
)
```

#### 激活消息

**位置**: claudeInChrome.ts:11-14

```typescript
const SKILL_ACTIVATION_MESSAGE = `
Now that this skill is invoked, you have access to Chrome browser automation tools.
IMPORTANT: Start by calling mcp__claude-in-chrome__tabs_context_mcp
`
```

---

### 3.14 scheduleRemoteAgents.ts - 远程代理调度 Skill

**文件位置**: `skills/bundled/scheduleRemoteAgents.ts`

#### Skill 基本信息

- **名称**: `schedule`
- **描述**: 创建、更新、列出或运行远程 Claude Code 代理
- **工具权限**: `[REMOTE_TRIGGER_TOOL_NAME, ASK_USER_QUESTION_TOOL_NAME]`
- **启用条件**: `getFeatureValue_CACHED_MAY_BE_STALE('tengu_surreal_dali', false) && isPolicyAllowed('allow_remote_sessions')`

#### Tagged ID 解码

**位置**: scheduleRemoteAgents.ts:23-57

```typescript
// Base58 解码 mcpsrv_ tagged ID 到 UUID
function taggedIdToUUID(taggedId: string): string | null {
  const prefix = 'mcpsrv_'
  if (!taggedId.startsWith(prefix)) return null

  const rest = taggedId.slice(prefix.length)
  const base58Data = rest.slice(2)  // Skip version prefix "01"

  // Decode base58 to bigint
  let n = 0n
  for (const c of base58Data) {
    const idx = BASE58.indexOf(c)
    if (idx === -1) return null
    n = n * 58n + BigInt(idx)
  }

  // Convert to UUID hex string
  const hex = n.toString(16).padStart(32, '0')
  return `${hex.slice(0, 8)}-${hex.slice(8, 12)}-...`
}
```

#### 环境创建

**位置**: scheduleRemoteAgents.ts:360-378

```typescript
// 如果没有环境，自动创建默认环境
if (environments.length === 0) {
  try {
    createdEnvironment = await createDefaultCloudEnvironment('claude-code-default')
    environments = [createdEnvironment]
  } catch (err) {
    // 失败处理...
  }
}
```

#### Cron 表达式示例

**位置**: scheduleRemoteAgents.ts:267-277

```typescript
// Cron 示例（用户本地时区转换到 UTC）：
// - 0 9 * * 1-5 — 每工作日 9am UTC
// - 0 */2 * * * — 每 2 小时
// - 0 0 * * * — 每天午夜 UTC
// - 30 14 * * 1 — 每周一 2:30pm UTC
```

---

## 4. Skill 条件加载机制

### 4.1 Feature Flag 控制

**位置**: bundledSkills.ts:35-79

```typescript
// Feature flag 条件加载模式：
if (feature('KAIROS') || feature('KAIROS_DREAM')) {
  const { registerDreamSkill } = require('./dream.js')
  registerDreamSkill()
}

// 常见 feature flags：
// - KAIROS / KAIROS_DREAM
// - REVIEW_ARTIFACT
// - AGENT_TRIGGERS
// - AGENT_TRIGGERS_REMOTE
// - BUILDING_CLAUDE_APPS
// - RUN_SKILL_GENERATOR
```

### 4.2 isEnabled 动态判断

**位置**: bundledSkills.ts:94

```typescript
// Command 对象的 isEnabled 字段：
isEnabled: definition.isEnabled

// 示例：
// - loop.ts: isEnabled: isKairosCronEnabled
// - remember.ts: isEnabled: () => isAutoMemoryEnabled()
// - keybindings.ts: isEnabled: isKeybindingCustomizationEnabled
// - claudeInChrome.ts: isEnabled: () => shouldAutoEnableClaudeInChrome()
// - scheduleRemoteAgents.ts: isEnabled: () => getFeatureValue && isPolicyAllowed
```

### 4.3 用户类型限制

**位置**: bundledSkills.ts:13-15

```typescript
if (process.env.USER_TYPE !== 'ant') {
  return  // 非 ant 用户跳过注册
}

// 仅 ant 用户可用的 Skills：
// - verify
// - skillify
// - remember
// - lorem-ipsum
// - stuck
```

### 4.4 动态 Skill 发现

**位置**: loadSkillsDir.ts:861-915

```typescript
async function discoverSkillDirsForPaths(filePaths, cwd): Promise<string[]> {
  const resolvedCwd = cwd.endsWith(pathSep) ? cwd.slice(0, -1) : cwd
  const newDirs: string[] = []

  for (const filePath of filePaths) {
    let currentDir = dirname(filePath)

    // Walk up to cwd but NOT including cwd itself
    while (currentDir.startsWith(resolvedCwd + pathSep)) {
      const skillDir = join(currentDir, '.claude', 'skills')

      if (!dynamicSkillDirs.has(skillDir)) {
        dynamicSkillDirs.add(skillDir)
        try {
          await fs.stat(skillDir)
          // Check gitignore before loading
          if (await isPathGitignored(currentDir, resolvedCwd)) {
            continue
          }
          newDirs.push(skillDir)
        } catch {
          // Directory doesn't exist
        }
      }

      // Move to parent
      const parent = dirname(currentDir)
      if (parent === currentDir) break
      currentDir = parent
    }
  }

  // Sort deepest first
  return newDirs.sort((a, b) => b.split(pathSep).length - a.split(pathSep).length)
}
```

### 4.5 条件 Skills（路径过滤）

**位置**: loadSkillsDir.ts:997-1058

```typescript
export function activateConditionalSkillsForPaths(filePaths, cwd): string[] {
  const activated: string[] = []

  for (const [name, skill] of conditionalSkills) {
    const skillIgnore = ignore().add(skill.paths)
    for (const filePath of filePaths) {
      const relativePath = relative(cwd, filePath)

      if (skillIgnore.ignores(relativePath)) {
        // Activate by moving to dynamic skills
        dynamicSkills.set(name, skill)
        conditionalSkills.delete(name)
        activatedConditionalSkillNames.add(name)
        activated.push(name)
        break
      }
    }
  }

  return activated
}
```

---

## 5. 从零实现指南

### 5.1 如何创建新 Bundled Skill

#### Step 1: 创建 Skill 文件

**位置**: `skills/bundled/mySkill.ts`

```typescript
import { registerBundledSkill } from '../bundledSkills.js'
import { logForDebugging } from '../../utils/debug.js'

export function registerMySkillSkill(): void {
  registerBundledSkill({
    name: 'my-skill',
    description: 'My skill description here',
    allowedTools: ['Read', 'Write', 'Bash'],
    argumentHint: '[argument description]',
    userInvocable: true,
    disableModelInvocation: false,  // 允许自动调用
    async getPromptForCommand(args, context) {
      // 1. 处理参数
      const trimmedArgs = args.trim()

      // 2. 检查前置条件
      if (!trimmedArgs) {
        return [{
          type: 'text',
          text: 'Please provide an argument.'
        }]
      }

      // 3. 构建 Prompt
      const prompt = `# My Skill

## User Request
${trimmedArgs}

## Instructions
1. Step one
2. Step two
3. Step three
`

      return [{ type: 'text', text: prompt }]
    },
  })
}
```

#### Step 2: 注册 Skill

**位置**: `skills/bundledSkills.ts`

```typescript
// 1. 导入注册函数
import { registerMySkillSkill } from './mySkill.js'

// 2. 在 initBundledSkills() 中调用
export function initBundledSkills(): void {
  registerMySkillSkill()  // 添加此行
  // ... 其他 Skills
}
```

#### Step 3: 条件注册（可选）

```typescript
// 如果需要 feature flag 控制：
if (feature('MY_FEATURE')) {
  const { registerMySkillSkill } = require('./mySkill.js')
  registerMySkillSkill()
}

// 如果需要动态启用条件：
registerBundledSkill({
  name: 'my-skill',
  isEnabled: () => isMyFeatureEnabled(),
  // ...
})

// 如果需要用户类型限制：
export function registerMySkillSkill(): void {
  if (process.env.USER_TYPE !== 'ant') {
    return
  }
  registerBundledSkill({ ... })
}
```

### 5.2 Skill 注册流程

**完整流程**:

1. **模块加载** → TypeScript 文件被 Bun 编译
2. **函数导出** → `registerMySkillSkill()` 函数导出
3. **初始化调用** → `initBundledSkills()` 调用注册函数
4. **Definition 构建** → 传入 `BundledSkillDefinition` 对象
5. **Command 创建** → `registerBundledSkill()` 构建 Command 对象
6. **Registry 存储** → Command 推入 `bundledSkills` 数组
7. **系统加载** → `getBundledSkills()` 返回所有注册的 Skills

### 5.3 Prompt 模板设计

#### 基础结构

```typescript
const SKILL_PROMPT = `# Skill Title

Brief description of what this skill does.

## Context
Current state or context information.

## Goal
What the skill should achieve.

## Steps

### 1. First Step
Detailed instructions for step one.

**Success criteria**: What proves this step is done.

### 2. Second Step
Detailed instructions for step two.

**Success criteria**: ...

## Rules
- Rule 1
- Rule 2

## User Request
{{userArgs}}
`
```

#### 动态内容生成

```typescript
async getPromptForCommand(args, context) {
  // 1. 获取动态数据
  const sessionMemory = await getSessionMemoryContent()
  const currentFiles = await readdir(getCwd())

  // 2. 构建动态部分
  const dynamicSection = `
## Current Files
${currentFiles.join('\n')}
`

  // 3. 组合 Prompt
  const fullPrompt = SKILL_PROMPT
    .replace('{{sessionMemory}}', sessionMemory)
    + dynamicSection

  if (args) {
    fullPrompt += `\n## User Request\n\n${args}`
  }

  return [{ type: 'text', text: fullPrompt }]
}
```

#### 参数处理模式

```typescript
// 1. 必需参数
if (!args.trim()) {
  return [{
    type: 'text',
    text: 'Usage: /my-skill <required-argument>'
  }]
}

// 2. 参数解析
const parts = args.split(' ')
const firstArg = parts[0]
const restArgs = parts.slice(1).join(' ')

// 3. 参数验证
const parsed = parseInt(firstArg)
if (isNaN(parsed)) {
  return [{
    type: 'text',
    text: 'Invalid argument. Expected a number.'
  }]
}
```

### 5.4 附加文件处理

```typescript
// 1. 定义附加文件
registerBundledSkill({
  name: 'my-skill',
  files: {
    'examples/example1.md': 'Example content...',
    'templates/template1.md': 'Template content...',
  },
  async getPromptForCommand(args) {
    // 文件会在首次调用时自动提取到磁盘
    // Prompt 会自动添加 "Base directory for this skill: <dir>" 前缀
    return [{ type: 'text', text: prompt }]
  },
})

// 2. 文件提取位置
// getBundledSkillExtractDir(skillName) 返回提取目录
// 默认位置: ~/.claude/bundled-skills/<skill-name>/
```

### 5.5 工具权限设计

```typescript
// 最小权限原则
allowedTools: ['Read']  // 只读

// 指定模式权限
allowedTools: [
  'Read',
  'Write(.claude)',  // 只能写入 .claude 目录
  'Bash(git:*)',      // 只能运行 git 命令
]

// 完全权限（谨慎使用）
allowedTools: ['Bash', 'Read', 'Write', 'Edit', 'Glob', 'Grep']

// 禁用特定工具（通过不添加到列表）
```

### 5.6 测试 Skill

```typescript
// 1. 手动测试
// 运行: claude
// 输入: /my-skill test-argument

// 2. 检查注册
import { getBundledSkills } from './bundledSkills.js'
const skills = getBundledSkills()
const mySkill = skills.find(s => s.name === 'my-skill')
console.log(mySkill)

// 3. 测试 Prompt 生成
const prompt = await mySkill.getPromptForCommand('test', mockContext)
console.log(prompt[0].text)
```

---

## 6. 最佳实践总结

### 6.1 Prompt 设计原则

1. **结构清晰**: 使用 Markdown 标题分层组织内容
2. **指令具体**: 明确的步骤说明，避免模糊描述
3. **成功标准**: 每个步骤定义完成标准
4. **错误处理**: 提供错误诊断和修复指引
5. **上下文丰富**: 包含必要的参考信息和示例

### 6.2 性能优化

1. **懒加载**: 大内容通过动态 import 加载（claudeApiContent.ts:7）
2. **尾部读取**: 大文件只读取必要部分（debug.ts:34-49）
3. **闭包记忆化**: 文件提取使用 Promise 级缓存（bundledSkills.ts:64）
4. **并发处理**: 多代理并行执行（simplify.ts:18）

### 6.3 安全考虑

1. **路径验证**: 防止路径遍历攻击（bundledSkills.ts:196-206）
2. **安全写入**: 使用 O_EXCL 和 O_NOFOLLOW 标志（bundledSkills.ts:176-194）
3. **权限限制**: 最小权限原则
4. **Gitignore 检查**: 动态发现前检查 gitignore（loadSkillsDir.ts:891-897）

### 6.4 用户体验

1. **参数提示**: 提供 argumentHint 指导用户
2. **使用时机**: whenToUse 帮助模型自动调用
3. **错误友好**: 清晰的错误消息和修复建议
4. **进度反馈**: 进度表格和状态更新（batch.ts:77-88）

---

## 附录：Skills 功能矩阵

| Skill | 用户类型 | Feature Flag | 工具权限 | 自动调用 |
|-------|----------|--------------|----------|----------|
| debug | all | - | Read, Grep, Glob | false |
| claude-api | all | BUILDING_CLAUDE_APPS | Read, Grep, Glob, WebFetch | true |
| skillify | ant | - | Read, Write, Edit, Glob, Grep, AskUserQuestion, Bash(mkdir:*) | false |
| verify | ant | - | - | true |
| batch | all | - | - | false |
| loop | all | AGENT_TRIGGERS | - | true |
| remember | ant | - | - | true |
| simplify | all | - | - | true |
| stuck | ant | - | - | true |
| update-config | all | - | Read | true |
| keybindings-help | all | - | Read | false |
| lorem-ipsum | ant | - | - | true |
| claude-in-chrome | all | auto-enable | mcp__claude-in-chrome__* | true |
| schedule | all | AGENT_TRIGGERS_REMOTE | RemoteTrigger, AskUserQuestion | true |

---

*分析完成: 2026-04-02*