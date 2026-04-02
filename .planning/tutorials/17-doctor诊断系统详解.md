# Doctor诊断系统详解

**教程编号:** 17
**分析日期:** 2026-04-02
**适用版本:** Claude Code CLI

---

## 1. 概述

### 1.1 Doctor命令的作用

`/doctor` 命令是Claude Code CLI的系统诊断工具，用于全面检查安装状态、配置正确性和运行环境。该命令提供了可视化的诊断界面，帮助用户快速定位问题并提供修复建议。

**核心功能：**
- 检测安装类型和版本信息
- 验证配置文件正确性
- 检查环境变量设置
- 监控MCP服务器配置状态
- 显示沙箱安全配置
- 检测Git仓库状态
- 验证权限配置规则
- 展示Agent系统状态

### 1.2 系统诊断范围

Doctor系统涵盖以下诊断维度：

| 诊断领域 | 检查内容 | 实现位置 |
|---------|---------|---------|
| **安装状态** | 安装类型、版本、路径 | `utils/doctorDiagnostic.ts` |
| **配置验证** | 设置文件语法、值合法性 | `components/ValidationErrorsList.tsx` |
| **MCP配置** | 服务器配置解析、连接状态 | `components/mcp/McpParsingWarnings.tsx` |
| **沙箱状态** | 依赖完整性、安全配置 | `components/sandbox/SandboxDoctorSection.tsx` |
| **环境变量** | 输出限制、token配置 | `screens/Doctor.tsx` |
| **上下文警告** | CLAUDE.md大小、Agent描述、权限规则 | `utils/doctorContextWarnings.ts` |
| **Git状态** | 仓库完整性、远程配置 | `utils/git.ts` |

---

## 2. commands/doctor/ 目录逐行分析

### 2.1 commands/doctor/index.ts - 命令注册

**文件位置:** `commands/doctor/index.ts`

```typescript
import type { Command } from '../../commands.js'
import { isEnvTruthy } from '../../utils/envUtils.js'

const doctor: Command = {
  name: 'doctor',
  description: 'Diagnose and verify your Claude Code installation and settings',
  isEnabled: () => !isEnvTruthy(process.env.DISABLE_DOCTOR_COMMAND),
  type: 'local-jsx',
  load: () => import('./doctor.js'),
}

export default doctor
```

**逐行解析：**

| 行号 | 代码 | 说明 |
|-----|------|------|
| 1 | `import type { Command }` | 导入命令类型定义 |
| 2 | `import { isEnvTruthy }` | 导入环境变量检查函数 |
| 4-10 | `const doctor: Command` | 定义命令配置对象 |
| 5 | `name: 'doctor'` | 命令名称，用户输入 `/doctor` 触发 |
| 6 | `description` | 命令描述，用于帮助文档 |
| 7 | `isEnabled()` | 动态启用检查，可通过环境变量禁用 |
| 8 | `type: 'local-jsx'` | 命令类型，表示返回React组件 |
| 9 | `load()` | 懒加载实现，减少启动开销 |

**关键设计：**

1. **环境变量控制:** `DISABLE_DOCTOR_COMMAND=true` 可禁用命令，用于特殊场景（如HFI环境）

2. **懒加载模式:** 使用动态import，只在命令执行时加载实现代码

3. **JSX命令类型:** `local-jsx` 类型命令返回React组件，由终端UI渲染系统显示

### 2.2 commands/doctor/doctor.tsx - 命令实现

**文件位置:** `commands/doctor/doctor.tsx`

```typescript
import React from 'react';
import { Doctor } from '../../screens/Doctor.js';
import type { LocalJSXCommandCall } from '../../types/command.js';

export const call: LocalJSXCommandCall = (onDone, _context, _args) => {
  return Promise.resolve(<Doctor onDone={onDone} />);
};
```

**逐行解析：**

| 行号 | 代码 | 说明 |
|-----|------|------|
| 1 | `import React` | React框架导入 |
| 2 | `import { Doctor }` | 导入诊断UI组件 |
| 3 | `import type { LocalJSXCommandCall }` | 导入JSX命令类型 |
| 5-6 | `export const call` | 命令实现函数 |
| 6 | `Promise.resolve(<Doctor ...>)` | 返回Doctor组件的Promise |

**设计特点：**

- **最小化实现:** 命令层仅负责渲染Doctor组件，逻辑全在screen层
- **onDone回调:** 用于命令完成后的清理，传递显示选项
- **Promise返回:** 符合命令系统的异步接口规范

---

## 3. screens/Doctor.tsx 分析

### 3.1 诊断UI组件架构

**文件位置:** `screens/Doctor.tsx`

Doctor组件是诊断系统的核心UI，采用React + Ink终端渲染架构。

**组件结构：**

```typescript
type Props = {
  onDone: (result?: string, options?: {
    display?: CommandResultDisplay;
  }) => void;
};

type AgentInfo = {
  activeAgents: Array<{
    agentType: string;
    source: SettingSource | 'built-in' | 'plugin';
  }>;
  userAgentsDir: string;
  projectAgentsDir: string;
  userDirExists: boolean;
  projectDirExists: boolean;
  failedFiles?: Array<{
    path: string;
    error: string;
  }>;
};

type VersionLockInfo = {
  enabled: boolean;
  locks: LockInfo[];
  locksDir: string;
  staleLocksCleaned: number;
};
```

### 3.2 主要诊断项列表

Doctor组件加载以下诊断数据：

```typescript
// 状态管理
const agentDefinitions = useAppState(state => state.agentDefinitions);
const mcpTools = useAppState(state => state.mcpTools);
const toolPermissionContext = useAppState(state => state.toolPermissionContext);
const pluginsErrors = useAppState(state => state.pluginsErrors);

// 诊断数据
const [diagnostic, setDiagnostic] = useState<DiagnosticInfo | null>(null);
const [agentInfo, setAgentInfo] = useState<AgentInfo | null>(null);
const [contextWarnings, setContextWarnings] = useState<ContextWarnings | null>(null);
const [versionLockInfo, setVersionLockInfo] = useState<VersionLockInfo | null>(null);
const validationErrors = useSettingsErrors();
```

### 3.3 结果显示结构

Doctor组件渲染以下诊断分区：

| 分区 | 组件 | 内容 |
|-----|------|------|
| **Diagnostics** | `<Text>Currently running: ...` | 安装类型、版本、路径 |
| **Updates** | `<Text>Auto-updates: ...` | 自动更新状态、权限 |
| **Versions** | `<DistTagsDisplay>` | npm版本标签（stable/latest） |
| **Invalid Settings** | `<ValidationErrorsList>` | 配置验证错误列表 |
| **Context Warnings** | 自定义渲染 | CLAUDE.md、Agent、MCP大小警告 |
| **Sandbox** | `<SandboxDoctorSection>` | 沙箱依赖检查 |
| **MCP Parsing Warnings** | `<McpParsingWarnings>` | MCP配置解析错误 |
| **Keybinding Warnings** | `<KeybindingWarnings>` | 快捷键冲突警告 |
| **Environment Variables** | 自定义渲染 | 环境变量验证错误 |
| **Version Locks** | 自定义渲染 | PID锁状态 |
| **Agent Parse Errors** | 自定义渲染 | Agent文件解析错误 |
| **Plugin Errors** | 自定义渲染 | 插件加载错误 |

### 3.4 加载流程分析

**初始化流程：**

```typescript
useEffect(() => {
  // 1. 获取诊断信息
  getDoctorDiagnostic().then(setDiagnostic);

  // 2. 获取Agent信息
  (async () => {
    const userAgentsDir = join(getClaudeConfigHomeDir(), "agents");
    const projectAgentsDir = join(getOriginalCwd(), ".claude", "agents");
    const { activeAgents, allAgents, failedFiles } = agentDefinitions;
    const [userDirExists, projectDirExists] = await Promise.all([
      pathExists(userAgentsDir),
      pathExists(projectAgentsDir)
    ]);
    setAgentInfo({ ... });

    // 3. 检查上下文警告
    const warnings = await checkContextWarnings(tools, {...}, async () => toolPermissionContext);
    setContextWarnings(warnings);

    // 4. 检查版本锁
    if (isPidBasedLockingEnabled()) {
      const locksDir = join(getXDGStateHome(), "claude", "locks");
      const staleLocksCleaned = cleanupStaleLocks(locksDir);
      const locks = getAllLockInfo(locksDir);
      setVersionLockInfo({ enabled: true, locks, locksDir, staleLocksCleaned });
    }
  })();
}, [toolPermissionContext, tools, agentDefinitions]);
```

---

## 4. 诊断检查项详解

### 4.1 安装状态诊断

**实现位置:** `utils/doctorDiagnostic.ts`

**核心函数:** `getDoctorDiagnostic()`

```typescript
export type DiagnosticInfo = {
  installationType: InstallationType    // 安装类型
  version: string                        // 版本号
  installationPath: string               // 安装路径
  invokedBinary: string                  // 调用的二进制
  configInstallMethod: InstallMethod | 'not set'  // 配置的安装方法
  autoUpdates: string                    // 自动更新状态
  hasUpdatePermissions: boolean | null   // 更新权限
  multipleInstallations: Array<{ type: string; path: string }>  // 多安装检测
  warnings: Array<{ issue: string; fix: string }>  // 警告列表
  recommendation?: string                // 推荐建议
  packageManager?: string                // 包管理器
  ripgrepStatus: {                       // ripgrep状态
    working: boolean
    mode: 'system' | 'builtin' | 'embedded'
    systemPath: string | null
  }
}
```

**安装类型检测：**

```typescript
export async function getCurrentInstallationType(): Promise<InstallationType> {
  if (process.env.NODE_ENV === 'development') {
    return 'development';
  }

  // 检查bundled模式
  if (isInBundledMode()) {
    // 检测包管理器安装
    if (detectHomebrew() || detectWinget() || detectMise() || ...) {
      return 'package-manager';
    }
    return 'native';
  }

  // 检查本地npm安装
  if (isRunningFromLocalInstallation()) {
    return 'npm-local';
  }

  // 检查npm全局安装路径
  const npmGlobalPaths = [
    '/usr/local/lib/node_modules',
    '/usr/lib/node_modules',
    '/opt/homebrew/lib/node_modules',
    ...
  ];

  if (npmGlobalPaths.some(path => invokedPath.includes(path))) {
    return 'npm-global';
  }

  return 'unknown';
}
```

**安装类型枚举：**

| 类型 | 说明 | 检测方法 |
|-----|------|---------|
| `npm-global` | npm全局安装 | 检查标准npm全局路径 |
| `npm-local` | npm本地安装 | `isRunningFromLocalInstallation()` |
| `native` | 原生打包安装 | `isInBundledMode()` |
| `package-manager` | 包管理器安装 | `detectHomebrew()`, `detectWinget()` |
| `development` | 开发模式运行 | `NODE_ENV === 'development'` |
| `unknown` | 无法识别 | 默认返回 |

**多安装检测：**

```typescript
async function detectMultipleInstallations(): Promise<Array<{ type: string; path: string }>> {
  const installations: Array<{ type: string; path: string }> = [];

  // 检查本地安装
  const localPath = join(homedir(), '.claude', 'local');
  if (await localInstallationExists()) {
    installations.push({ type: 'npm-local', path: localPath });
  }

  // 检查全局npm安装
  const npmResult = await execFileNoThrow('npm', ['-g', 'config', 'get', 'prefix']);
  if (npmResult.code === 0 && npmResult.stdout) {
    const npmPrefix = npmResult.stdout.trim();
    const globalBinPath = join(npmPrefix, 'bin', 'claude');
    // 检查bin/claude是否存在
    // 检查是否是Homebrew cask
    // 添加到installations列表
  }

  // 检查原生安装
  const nativeBinPath = join(homedir(), '.local', 'bin', 'claude');
  // 检查路径存在性

  return installations;
}
```

**配置问题检测：**

```typescript
async function detectConfigurationIssues(type: InstallationType): Promise<Array<{ issue: string; fix: string }>> {
  const warnings: Array<{ issue: string; fix: string }> = [];

  // 检查managed-settings.json的strictPluginOnlyCustomization字段
  // 检查PATH中是否包含~/.local/bin（对于native安装）
  // 检查配置的installMethod与实际类型是否匹配
  // 检查alias是否有效

  return warnings;
}
```

### 4.2 Git配置检查

**实现位置:** `utils/git.ts`

**核心函数：**

| 函数 | 作用 | 返回值 |
|-----|------|-------|
| `findGitRoot(startPath)` | 查找.git目录位置 | `string | null` |
| `getIsGit()` | 检查当前目录是否在Git仓库 | `Promise<boolean>` |
| `getHead()` | 获取HEAD commit SHA | `Promise<string>` |
| `getBranch()` | 获取当前分支名 | `Promise<string>` |
| `getRemoteUrl()` | 获取远程URL | `Promise<string | null>` |
| `getIsClean()` | 检查仓库是否干净 | `Promise<boolean>` |
| `getGitState()` | 获取完整Git状态 | `Promise<GitRepoState | null>` |

**Git根目录查找算法：**

```typescript
const findGitRootImpl = memoizeWithLRU(
  (startPath: string): string | typeof GIT_ROOT_NOT_FOUND => {
    let current = resolve(startPath);
    const root = current.substring(0, current.indexOf(sep) + 1) || sep;
    let statCount = 0;

    while (current !== root) {
      try {
        const gitPath = join(current, '.git');
        statCount++;
        const stat = statSync(gitPath);
        // .git可以是目录（普通仓库）或文件（worktree/submodule）
        if (stat.isDirectory() || stat.isFile()) {
          return current.normalize('NFC');
        }
      } catch {
        // .git不存在，继续向上查找
      }
      current = dirname(current);
    }
    return GIT_ROOT_NOT_FOUND;
  },
  path => path,
  50  // LRU缓存大小
);
```

**安全特性：**

1. **Worktree支持:** `.git`文件指向worktree的gitdir
2. **Canonical root解析:** `resolveCanonicalRoot()` 处理worktree指向主仓库
3. **安全验证:** 检查commondir和gitdir回链，防止恶意仓库攻击
4. **Bare repo检测:** `isCurrentDirectoryBareGitRepo()` 检测目录是否伪装成bare repo

**安全验证代码（防止恶意仓库攻击）：**

```typescript
const resolveCanonicalRoot = memoizeWithLRU(
  (gitRoot: string): string => {
    // 读取.git文件内容
    const gitContent = readFileSync(join(gitRoot, '.git'), 'utf-8').trim();
    if (!gitContent.startsWith('gitdir:')) {
      return gitRoot;
    }

    // 解析worktree路径
    const worktreeGitDir = resolve(gitRoot, gitContent.slice('gitdir:'.length).trim());

    // 解析commondir指向的共享.git目录
    const commonDir = resolve(worktreeGitDir, readFileSync(join(worktreeGitDir, 'commondir'), 'utf-8').trim());

    // SECURITY: 验证结构符合git worktree add创建的格式
    // 1. worktreeGitDir必须是<commonDir>/worktrees/的直接子目录
    if (resolve(dirname(worktreeGitDir)) !== join(commonDir, 'worktrees')) {
      return gitRoot;  // 验证失败，返回原路径
    }

    // 2. gitdir文件必须指向回<gitRoot>/.git
    const backlink = realpathSync(readFileSync(join(worktreeGitDir, 'gitdir'), 'utf-8').trim());
    if (backlink !== join(realpathSync(gitRoot), '.git')) {
      return gitRoot;  // 验证失败
    }

    // 返回canonical root（主仓库路径）
    return dirname(commonDir).normalize('NFC');
  },
  root => root,
  50
);
```

### 4.3 认证状态检查

**实现位置:** `utils/auth.ts`

**认证源检测：**

```typescript
export function getAuthTokenSource() {
  // --bare模式：只允许apiKeyHelper
  if (isBareMode()) {
    if (getConfiguredApiKeyHelper()) {
      return { source: 'apiKeyHelper' as const, hasToken: true };
    }
    return { source: 'none' as const, hasToken: false };
  }

  // 检查环境变量token
  if (process.env.ANTHROPIC_AUTH_TOKEN && !isManagedOAuthContext()) {
    return { source: 'ANTHROPIC_AUTH_TOKEN' as const, hasToken: true };
  }

  // 检查OAuth token
  if (process.env.CLAUDE_CODE_OAUTH_TOKEN) {
    return { source: 'CLAUDE_CODE_OAUTH_TOKEN' as const, hasToken: true };
  }

  // 检查文件描述符token
  const oauthTokenFromFd = getOAuthTokenFromFileDescriptor();
  if (oauthTokenFromFd) {
    if (process.env.CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR) {
      return { source: 'CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR' as const, hasToken: true };
    }
    return { source: 'CCR_OAUTH_TOKEN_FILE' as const, hasToken: true };
  }

  // 检查apiKeyHelper配置
  const apiKeyHelper = getConfiguredApiKeyHelper();
  if (apiKeyHelper && !isManagedOAuthContext()) {
    return { source: 'apiKeyHelper' as const, hasToken: true };
  }

  // 检查keychain中的OAuth token
  const oauthTokens = getClaudeAIOAuthTokens();
  if (oauthTokens) {
    return { source: 'claudeai_oauth' as const, hasToken: true };
  }

  return { source: 'none' as const, hasToken: false };
}
```

**Anthropic认证启用判断：**

```typescript
export function isAnthropicAuthEnabled(): boolean {
  // --bare模式：禁用OAuth
  if (isBareMode()) return false;

  // SSH远程代理模式
  if (process.env.ANTHROPIC_UNIX_SOCKET) {
    return !!process.env.CLAUDE_CODE_OAUTH_TOKEN;
  }

  // 检查第三方服务（Bedrock/Vertex/Foundry）
  const is3P = isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK) ||
               isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX) ||
               isEnvTruthy(process.env.CLAUDE_CODE_USE_FOUNDRY);

  // 检查外部API key源
  const { source: apiKeySource } = getAnthropicApiKeyWithSource({...});
  const hasExternalApiKey = apiKeySource === 'ANTHROPIC_API_KEY' || apiKeySource === 'apiKeyHelper';
  const hasExternalAuthToken = process.env.ANTHROPIC_AUTH_TOKEN || apiKeyHelper || ...;

  // 禁用Anthropic auth的条件
  const shouldDisableAuth = is3P ||
    (hasExternalAuthToken && !isManagedOAuthContext()) ||
    (hasExternalApiKey && !isManagedOAuthContext());

  return !shouldDisableAuth;
}
```

### 4.4 上下文警告检查

**实现位置:** `utils/doctorContextWarnings.ts`

**警告类型：**

```typescript
export type ContextWarning = {
  type:
    | 'claudemd_files'       // CLAUDE.md文件过大
    | 'agent_descriptions'   // Agent描述占用过多token
    | 'mcp_tools'            // MCP工具占用过多token
    | 'unreachable_rules'    // 不可达的权限规则
  severity: 'warning' | 'error'
  message: string
  details: string[]
  currentValue: number
  threshold: number
};
```

**CLAUDE.md文件检查：**

```typescript
async function checkClaudeMdFiles(): Promise<ContextWarning | null> {
  const largeFiles = getLargeMemoryFiles(await getMemoryFiles());

  if (largeFiles.length === 0) {
    return null;
  }

  const details = largeFiles
    .sort((a, b) => b.content.length - a.content.length)
    .map(file => `${file.path}: ${file.content.length.toLocaleString()} chars`);

  const message = largeFiles.length === 1
    ? `Large CLAUDE.md file detected (${largeFiles[0]!.content.length.toLocaleString()} chars > ${MAX_MEMORY_CHARACTER_COUNT.toLocaleString()})`
    : `${largeFiles.length} large CLAUDE.md files detected (each > ${MAX_MEMORY_CHARACTER_COUNT.toLocaleString()} chars)`;

  return {
    type: 'claudemd_files',
    severity: 'warning',
    message,
    details,
    currentValue: largeFiles.length,
    threshold: MAX_MEMORY_CHARACTER_COUNT,
  };
}
```

**阈值配置：**

| 检查项 | 阈值 | 说明 |
|-------|-----|------|
| `MAX_MEMORY_CHARACTER_COUNT` | 40,000 chars | 单个CLAUDE.md文件最大字符数 |
| `AGENT_DESCRIPTIONS_THRESHOLD` | 15,000 tokens | Agent描述总token阈值 |
| `MCP_TOOLS_THRESHOLD` | 25,000 tokens | MCP工具总token阈值 |

**Agent描述检查：**

```typescript
async function checkAgentDescriptions(agentInfo: AgentDefinitionsResult | null): Promise<ContextWarning | null> {
  if (!agentInfo) return null;

  const totalTokens = getAgentDescriptionsTotalTokens(agentInfo);

  if (totalTokens <= AGENT_DESCRIPTIONS_THRESHOLD) {
    return null;
  }

  // 计算每个Agent的token数
  const agentTokens = agentInfo.activeAgents
    .filter(a => a.source !== 'built-in')
    .map(agent => {
      const description = `${agent.agentType}: ${agent.whenToUse}`;
      return {
        name: agent.agentType,
        tokens: roughTokenCountEstimation(description),
      };
    })
    .sort((a, b) => b.tokens - a.tokens);

  const details = agentTokens.slice(0, 5).map(agent => `${agent.name}: ~${agent.tokens.toLocaleString()} tokens`);
  if (agentTokens.length > 5) {
    details.push(`(${agentTokens.length - 5} more custom agents)`);
  }

  return {
    type: 'agent_descriptions',
    severity: 'warning',
    message: `Large agent descriptions (~${totalTokens.toLocaleString()} tokens > ${AGENT_DESCRIPTIONS_THRESHOLD.toLocaleString()})`,
    details,
    currentValue: totalTokens,
    threshold: AGENT_DESCRIPTIONS_THRESHOLD,
  };
}
```

**不可达权限规则检查：**

```typescript
async function checkUnreachableRules(
  getToolPermissionContext: () => Promise<ToolPermissionContext>,
): Promise<ContextWarning | null> {
  const context = await getToolPermissionContext();
  const sandboxAutoAllowEnabled =
    SandboxManager.isSandboxingEnabled() &&
    SandboxManager.isAutoAllowBashIfSandboxedEnabled();

  const unreachable = detectUnreachableRules(context, { sandboxAutoAllowEnabled });

  if (unreachable.length === 0) {
    return null;
  }

  const details = unreachable.flatMap(r => [
    `${permissionRuleValueToString(r.rule.ruleValue)}: ${r.reason}`,
    `  Fix: ${r.fix}`,
  ]);

  return {
    type: 'unreachable_rules',
    severity: 'warning',
    message: `${unreachable.length} ${plural(unreachable.length, 'unreachable permission rule')} detected`,
    details,
    currentValue: unreachable.length,
    threshold: 0,
  };
}
```

---

## 5. 相关诊断工具

### 5.1 验证错误列表组件

**文件位置:** `components/ValidationErrorsList.tsx`

**功能:** 将配置验证错误以树形结构展示

**核心逻辑：**

```typescript
function buildNestedTree(errors: ValidationError[]): TreeNode {
  const tree: TreeNode = {};

  errors.forEach(error => {
    if (!error.path) {
      // 根级别错误
      tree[''] = error.message;
      return;
    }

    // 尝试增强路径显示
    const pathParts = error.path.split('.');
    let modifiedPath = error.path;

    // 如果有无效值，让路径更可读
    if (error.invalidValue !== null && error.invalidValue !== undefined && pathParts.length > 0) {
      const newPathParts: string[] = [];
      for (let i = 0; i < pathParts.length; i++) {
        const part = pathParts[i];
        if (!part) continue;
        const numericPart = parseInt(part, 10);

        // 如果是数字索引且是最后一部分，显示值
        if (!isNaN(numericPart) && i === pathParts.length - 1) {
          let displayValue: string;
          if (typeof error.invalidValue === 'string') {
            displayValue = `"${error.invalidValue}"`;
          } else if (error.invalidValue === null) {
            displayValue = 'null';
          } else if (error.invalidValue === undefined) {
            displayValue = 'undefined';
          } else {
            displayValue = String(error.invalidValue);
          }
          newPathParts.push(displayValue);
        } else {
          newPathParts.push(part);
        }
      }
      modifiedPath = newPathParts.join('.');
    }
    setWith(tree, modifiedPath, error.message, Object);
  });

  return tree;
}
```

**展示效果示例：**

```
~/.claude/settings.json
  └ allowedTools
      └ "Bash(npm run *)" - 'npm run *' is too broad. Use specific commands.
      └ "Read(*)" - Read(*) allows reading any file. Consider restricting.
```

### 5.2 MCP解析警告组件

**文件位置:** `components/mcp/McpParsingWarnings.tsx`

**功能:** 显示MCP服务器配置的解析错误和警告

**核心逻辑：**

```typescript
export function McpParsingWarnings(): React.ReactNode {
  // 获取各scope的MCP配置
  const scopes = useMemo(
    () => [
      { scope: 'user', config: getMcpConfigsByScope('user') },
      { scope: 'project', config: getMcpConfigsByScope('project') },
      { scope: 'local', config: getMcpConfigsByScope('local') },
      { scope: 'enterprise', config: getMcpConfigsByScope('enterprise') },
    ],
    []
  );

  const hasParsingErrors = scopes.some(
    ({ config }) => filterErrors(config.errors, 'fatal').length > 0,
  );
  const hasWarnings = scopes.some(
    ({ config }) => filterErrors(config.errors, 'warning').length > 0,
  );

  if (!hasParsingErrors && !hasWarnings) {
    return null;
  }

  return (
    <Box flexDirection="column" marginTop={1} marginBottom={1}>
      <Text bold>MCP Config Diagnostics</Text>
      <Box marginTop={1}>
        <Text dimColor>
          For help configuring MCP servers, see: {' '}
          <Link url="https://code.claude.com/docs/en/mcp">
            https://code.claude.com/docs/en/mcp
          </Link>
        </Text>
      </Box>
      {scopes.map(({ scope, config }) => (
        <McpConfigErrorSection
          key={scope}
          scope={scope}
          parsingErrors={filterErrors(config.errors, 'fatal')}
          warnings={filterErrors(config.errors, 'warning')}
        />
      ))}
    </Box>
  );
}
```

**错误过滤：**

```typescript
function filterErrors(
  errors: ValidationError[],
  severity: 'fatal' | 'warning'
): ValidationError[] {
  return errors.filter(e => e.mcpErrorMetadata?.severity === severity);
}
```

**显示效果示例：**

```
MCP Config Diagnostics
  For help configuring MCP servers, see: https://code.claude.com/docs/en/mcp

  [Failed to parse] User
  Location: ~/.claude/settings.json
    └ [Error] [weather-server] command: Command not found: "weather-cli"
    └ [Warning] [database-server] args: Using deprecated 'args' format

  [Contains warnings] Project
  Location: .claude/settings.json
    └ [Warning] [git-server] env: Environment variable API_KEY not set
```

### 5.3 沙箱诊断组件

**文件位置:** `components/sandbox/SandboxDoctorSection.tsx`

**功能:** 检查沙箱依赖和配置状态

**核心逻辑：**

```typescript
export function SandboxDoctorSection(): React.ReactNode {
  // 平台不支持沙箱则不显示
  if (!SandboxManager.isSupportedPlatform()) {
    return null;
  }

  // 设置中未启用沙箱则不显示
  if (!SandboxManager.isSandboxEnabledInSettings()) {
    return null;
  }

  const depCheck = SandboxManager.checkDependencies();
  const hasErrors = depCheck.errors.length > 0;
  const hasWarnings = depCheck.warnings.length > 0;

  // 无错误和警告则不显示
  if (!hasErrors && !hasWarnings) {
    return null;
  }

  const statusColor = hasErrors ? "error" : "warning";
  const statusText = hasErrors ? "Missing dependencies" : "Available (with warnings)";

  return (
    <Box flexDirection="column">
      <Text bold>Sandbox</Text>
      <Text>
        └ Status: <Text color={statusColor}>{statusText}</Text>
      </Text>
      {depCheck.errors.map(e => (
        <Text key={e} color="error">└ {e}</Text>
      ))}
      {depCheck.warnings.map(w => (
        <Text key={w} color="warning">└ {w}</Text>
      ))}
      {hasErrors && (
        <Text dimColor>└ Run /sandbox for install instructions</Text>
      )}
    </Box>
  );
}
```

**显示效果示例：**

```
Sandbox
  └ Status: Missing dependencies
  └ Missing: landlock
  └ Missing: bwrap
  └ Run /sandbox for install instructions
```

---

## 6. 从零实现指南

### 6.1 如何设计诊断命令

**步骤1: 定义命令注册**

```typescript
// commands/doctor/index.ts
import type { Command } from '../../commands.js';

const doctor: Command = {
  name: 'doctor',
  description: '系统诊断工具',
  isEnabled: () => true,  // 或环境变量控制
  type: 'local-jsx',      // 返回React组件
  load: () => import('./doctor.js'),
};

export default doctor;
```

**步骤2: 实现命令调用**

```typescript
// commands/doctor/doctor.tsx
import React from 'react';
import { Doctor } from '../../screens/Doctor.js';
import type { LocalJSXCommandCall } from '../../types/command.js';

export const call: LocalJSXCommandCall = (onDone, _context, _args) => {
  return Promise.resolve(<Doctor onDone={onDone} />);
};
```

### 6.2 检查项实现

**步骤3: 创建诊断数据收集函数**

```typescript
// utils/doctorDiagnostic.ts

// 定义诊断信息类型
export type DiagnosticInfo = {
  installationType: string;
  version: string;
  warnings: Array<{ issue: string; fix: string }>;
  // ...其他字段
};

// 实现诊断函数
export async function getDoctorDiagnostic(): Promise<DiagnosticInfo> {
  const installationType = await detectInstallationType();
  const version = getVersion();
  const warnings = await collectWarnings();

  return {
    installationType,
    version,
    warnings,
    // ...
  };
}
```

**步骤4: 实现具体检查函数**

```typescript
// 检查安装类型
async function detectInstallationType(): Promise<string> {
  if (isBundledMode()) {
    return 'native';
  }
  if (isRunningFromLocalInstallation()) {
    return 'npm-local';
  }
  // ...更多检测逻辑
  return 'unknown';
}

// 收集警告信息
async function collectWarnings(): Promise<Array<{ issue: string; fix: string }>> {
  const warnings = [];

  // 检查PATH配置
  if (!isInPath()) {
    warnings.push({
      issue: '安装路径不在PATH中',
      fix: '添加路径到PATH环境变量',
    });
  }

  // 检查配置文件
  const configErrors = await checkConfigFiles();
  warnings.push(...configErrors);

  return warnings;
}
```

### 6.3 UI展示实现

**步骤5: 创建诊断UI组件**

```typescript
// screens/Doctor.tsx
import React, { useState, useEffect } from 'react';
import { Box, Text } from '../ink.js';
import { getDoctorDiagnostic } from '../utils/doctorDiagnostic.js';

export function Doctor({ onDone }: { onDone: () => void }) {
  const [diagnostic, setDiagnostic] = useState<DiagnosticInfo | null>(null);

  useEffect(() => {
    getDoctorDiagnostic().then(setDiagnostic);
  }, []);

  if (!diagnostic) {
    return <Text dimColor>正在检查系统状态...</Text>;
  }

  return (
    <Box flexDirection="column">
      <Text bold>诊断结果</Text>
      <Text>└ 安装类型: {diagnostic.installationType}</Text>
      <Text>└ 版本: {diagnostic.version}</Text>

      {diagnostic.warnings.length > 0 && (
        <Box flexDirection="column" marginTop={1}>
          <Text bold color="warning">警告</Text>
          {diagnostic.warnings.map(w => (
            <Box key={w.issue} flexDirection="column">
              <Text color="warning">└ 问题: {w.issue}</Text>
              <Text dimColor>  └ 修复: {w.fix}</Text>
            </Box>
          ))}
        </Box>
      )}
    </Box>
  );
}
```

**步骤6: 实现退出处理**

```typescript
// 添加按键绑定
useKeybindings({
  'confirm:yes': () => onDone(),
  'confirm:no': () => onDone(),
}, { context: 'Confirmation' });

// 显示退出提示
<Box>
  <PressEnterToContinue />
</Box>
```

---

## 7. 设计要点总结

### 7.1 架构设计原则

| 原则 | 实现方式 | 优势 |
|-----|---------|------|
| **分层架构** | command → screen → utils | 清晰职责分离 |
| **懒加载** | 动态import命令实现 | 减少启动开销 |
| **异步数据收集** | useEffect + Promise | 不阻塞UI渲染 |
| **LRU缓存** | git.ts使用memoizeWithLRU | 优化重复查询 |
| **安全验证** | Git worktree链验证 | 防止恶意仓库攻击 |

### 7.2 UI展示原则

| 原则 | 实现方式 | 效果 |
|-----|---------|------|
| **树形展示** | treeify函数 | 清晰层级结构 |
| **颜色区分** | error/warning/inactive | 快速识别问题级别 |
| **细节折叠** | 显示前5项+剩余计数 | 避免信息过载 |
| **修复建议** | 每个警告附带fix字段 | 直接可操作 |

### 7.3 扩展建议

**新增诊断项步骤：**

1. 在`utils/doctorDiagnostic.ts`中添加检查函数
2. 在`DiagnosticInfo`类型中添加新字段
3. 在`screens/Doctor.tsx`中添加渲染逻辑
4. 如需专用组件，创建在`components/`目录

**新增警告类型：**

1. 在`utils/doctorContextWarnings.ts`中扩展`ContextWarning.type`
2. 实现对应的check函数
3. 在`checkContextWarnings()`中调用新检查函数

---

## 8. 关键文件路径索引

| 文件 | 作用 |
|-----|------|
| `commands/doctor/index.ts` | 命令注册配置 |
| `commands/doctor/doctor.tsx` | 命令实现入口 |
| `screens/Doctor.tsx` | 诊断UI主组件 |
| `utils/doctorDiagnostic.ts` | 诊断数据收集 |
| `utils/doctorContextWarnings.ts` | 上下文警告检查 |
| `utils/git.ts` | Git状态检查 |
| `utils/auth.ts` | 认证状态检查 |
| `components/ValidationErrorsList.tsx` | 验证错误展示 |
| `components/mcp/McpParsingWarnings.tsx` | MCP配置警告 |
| `components/sandbox/SandboxDoctorSection.tsx` | 沙箱诊断组件 |

---

## 附录：诊断输出示例

```
Diagnostics
  └ Currently running: native (v1.0.35)
  └ Path: /home/user/.local/bin/claude
  └ Invoked: /home/user/.local/bin/claude
  └ Config install method: native
  └ Search: OK (bundled)

Updates
  └ Auto-updates: enabled
  └ Update permissions: Yes
  └ Auto-update channel: latest
  └ Latest version: 1.0.36
  └ Stable version: 1.0.35

Invalid Settings
  ~/.claude/settings.json
    └ allowedTools
        └ "Bash(*)" - Bash(*) allows running any command

Context Warnings
  └ Large CLAUDE.md file detected (45,000 chars > 40,000)
      └ /project/CLAUDE.md: 45,000 chars

Sandbox
  └ Status: Available (with warnings)
  └ landlock version 1.0 is outdated (recommended: 1.1+)

MCP Config Diagnostics
  └ For help configuring MCP servers, see: https://code.claude.com/docs/en/mcp

  [Contains warnings] User
  └ Location: ~/.claude/settings.json
      └ [Warning] [weather-server] timeout: Timeout exceeds recommended maximum

Version Locks
  └ No active version locks

Agent Parse Errors
  └ Failed to parse 2 agent file(s):
      └ custom-agent.md: Invalid YAML frontmatter
      └ helper.md: Missing required 'whenToUse' field

Plugin Errors
  └ 1 plugin error(s) detected:
      └ my-plugin: Failed to load: Module not found
```

---

*教程完成 - Doctor诊断系统详解*