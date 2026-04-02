# Feature Flag系统详解

**分析日期:** 2026-04-02

## 1. Feature Flag概述

Claude Code实现了两层Feature Flag系统：

### 1.1 编译时Feature Flag (bun:bundle feature)

使用Bun打包器的条件编译能力，在构建时决定哪些代码进入最终bundle。

```typescript
import { feature } from 'bun:bundle'

// 编译时判断 - 不会进入外部构建
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null
```

**核心特性：**
- **死代码消除（DCE）**：未启用的feature分支代码完全从bundle中移除
- **零运行时开销**：判断在编译时完成，无运行时条件检查
- **字符串隐藏**：敏感名称（如内部代号）不会出现在外部构建中

### 1.2 运行时Feature Flag (GrowthBook)

基于GrowthBook SDK的远程配置系统，支持动态特性开关。

```typescript
import { getFeatureValue_CACHED_MAY_BE_STALE } from '../services/analytics/growthbook.js'

// 运行时判断 - 可动态调整
const isEnabled = getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_flint', false)
```

**核心特性：**
- **动态控制**：无需重新部署即可调整feature状态
- **用户分桶**：支持按用户属性进行A/B测试
- **磁盘缓存**：启动时无需等待网络请求

---

## 2. bun:bundle feature()详解

### 2.1 基本用法

`feature()`函数由Bun打包器提供，在编译时返回布尔值：

```typescript
// commands.ts:59-122
import { feature } from 'bun:bundle'

// 条件导入 - 仅在feature启用时加载模块
const proactive =
  feature('PROACTIVE') || feature('KAIROS')
    ? require('./commands/proactive.js').default
    : null

const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null

const buddy = feature('BUDDY')
  ? require('./commands/buddy/index.js').default
  : null
```

### 2.2 Feature定义位置

Feature flags在构建配置中定义（通过`--define`参数）：

```typescript
// constants/keys.ts:5-11 - 根据USER_TYPE选择不同的GrowthBook SDK Key
export function getGrowthBookClientKey(): string {
  return process.env.USER_TYPE === 'ant'
    ? isEnvTruthy(process.env.ENABLE_GROWTHBOOK_DEV)
      ? 'sdk-yZQvlplybuXjYh6L'  // Ant开发环境
      : 'sdk-xRVcrliHIlrg4og4'  // Ant生产环境
    : 'sdk-zAZezfDKGoZuXXKe'    // 外部用户
}
```

**USER_TYPE是构建时定义：**
```typescript
// 构建命令示例（推测）
bun build ./entrypoints/cli.tsx \
  --define USER_TYPE='external' \
  --define feature.VOICE_MODE=false \
  --define feature.BUDDY=false \
  ...
```

### 2.3 常见编译时Feature Flags

| Feature Flag | 用途 | 外部构建状态 |
|--------------|------|-------------|
| `VOICE_MODE` | 语音模式 | `false` |
| `BUDDY` | Tamagotchi伴侣系统 | `false` |
| `KAIROS` | Assistant模式 | `false` |
| `BRIDGE_MODE` | 远程桥接模式 | `false` |
| `PROACTIVE` | 主动建议 | `false` |
| `DAEMON` | 后台守护进程 | `false` |
| `ULTRAPLAN` | 超级规划模式 | `false` |
| `FORK_SUBAGENT` | 子代理派生 | `false` |
| `WORKFLOW_SCRIPTS` | 工作流脚本 | `false` |
| `TRANSCRIPT_CLASSIFIER` | 对话分类器 | `false` |
| `DUMP_SYSTEM_PROMPT` | 系统提示导出 | `false` |

---

## 3. 条件编译实现

### 3.1 条件导入模式

使用`require()`配合三元运算符实现条件导入：

```typescript
// commands.ts:60-123 - Dead code elimination: conditional imports
/* eslint-disable @typescript-eslint/no-require-imports */
const proactive =
  feature('PROACTIVE') || feature('KAIROS')
    ? require('./commands/proactive.js').default
    : null

const bridge = feature('BRIDGE_MODE')
  ? require('./commands/bridge/index.js').default
    : null

const buddy = feature('BUDDY')
  ? (
      require('./commands/buddy/index.js') as typeof import('./commands/buddy/index.js')
    ).default
  : null
/* eslint-enable @typescript-eslint/no-require-imports */
```

**关键技术点：**
1. 使用`require()`而非`import` - 允许动态条件
2. 类型断言保持类型安全
3. ESLint禁用规则仅在条件导入块内生效

### 3.2 条件导出Beta Headers

```typescript
// constants/betas.ts:23-28
export const SUMMARIZE_CONNECTOR_TEXT_BETA_HEADER = feature('CONNECTOR_TEXT')
  ? 'summarize-connector-text-2026-03-13'
  : ''  // 外部构建返回空字符串

export const AFK_MODE_BETA_HEADER = feature('TRANSCRIPT_CLASSIFIER')
  ? 'afk-mode-2026-01-31'
  : ''

export const CLI_INTERNAL_BETA_HEADER =
  process.env.USER_TYPE === 'ant' ? 'cli-internal-2026-02-09' : ''
```

### 3.3 条件配置导入

```typescript
// constants/prompts.ts:64-98 - Dead code elimination: conditional imports for feature-gated modules
/* eslint-disable @typescript-eslint/no-require-imports */
const getCachedMCConfigForFRC = feature('CACHED_MICROCOMPACT')
  ? (
      require('../services/compact/cachedMCConfig.js') as typeof import('../services/compact/cachedMCConfig.js')
    ).getCachedMCConfig
  : null

const proactiveModule =
  feature('PROACTIVE') || feature('KAIROS')
    ? require('../proactive/index.js')
    : null

const briefToolModule =
  feature('KAIROS') || feature('KAIROS_BRIEF')
    ? (require('../tools/BriefTool/BriefTool.js') as typeof import('../tools/BriefTool/BriefTool.js'))
    : null
/* eslint-enable @typescript-eslint/no-require-imports */
```

### 3.4 条件执行块

```typescript
// entrypoints/cli.tsx:19-26 - Harness-science L0 ablation baseline
if (feature('ABLATION_BASELINE') && process.env.CLAUDE_CODE_ABLATION_BASELINE) {
  for (const k of ['CLAUDE_CODE_SIMPLE', 'CLAUDE_CODE_DISABLE_THINKING', ...]) {
    process.env[k] ??= '1';
  }
}

// entrypoints/cli.tsx:53-71 - Dump system prompt (ant-only)
if (feature('DUMP_SYSTEM_PROMPT') && args[0] === '--dump-system-prompt') {
  const { getSystemPrompt } = await import('../constants/prompts.js');
  const prompt = await getSystemPrompt([], model);
  console.log(prompt.join('\n'));
  return;
}
```

---

## 4. 死代码消除（DCE）

### 4.1 DCE原理

Bun打包器在编译时评估`feature()`调用：
- 当`feature('X')`定义为`false`时，三元表达式的`false`分支被完全移除
- 字符串字面量（如内部代号）不会出现在bundle中

```typescript
// 外部构建中，以下代码会被消除：
const buddy = feature('BUDDY')
  ? require('./commands/buddy/index.js').default
  : null

// 消除后变为：
const buddy = null
// buddy/index.js模块不会被打包
// "buddy"字符串不会出现在bundle中（防止搜索发现）
```

### 4.2 DCE注释标记

代码中明确标记DCE意图：

```typescript
// commands.ts:60
// Dead code elimination: conditional imports

// constants/prompts.ts:64
// Dead code elimination: conditional imports for feature-gated modules

// constants/prompts.ts:617
// DCE: `process.env.USER_TYPE === 'ant'` is build-time --define. It MUST be
// constant-fold it to `false` in external builds and eliminate the branch.

// screens/REPL.tsx:96
// Dead code elimination: conditional imports
```

### 4.3 DCE复杂度悬崖（Complexity Cliff）

```typescript
// tools/BashTool/bashPermissions.ts:81-89
// DCE cliff: Bun's feature() evaluator has a per-function complexity budget.
// bashToolHasPermission is right at the limit. `import { X as Y }` aliases
// inside the import block count toward this budget; when they push it over
// the threshold Bun can no longer prove feature('BASH_CLASSIFIER') is a
// constant and silently evaluates the ternaries to `false`, dropping every
// pendingClassifierCheck spread. Keep aliases as top-level const rebindings
// instead.

const bashCommandIsSafeAsync = bashCommandIsSafeAsync_DEPRECATED
const splitCommand = splitCommand_DEPRECATED
```

**关键注意：**
- Bun的`feature()`评估器有每函数复杂度预算
- 过多的import别名会超出预算，导致DCE失效
- 解决方案：使用顶层`const`重新绑定而非import别名

### 4.4 正向三元模式（Positive Ternary Pattern）

```typescript
// voice/voiceModeEnabled.ts:16-23
// Positive ternary pattern — see docs/feature-gating.md.
// Negative pattern (if (!feature(...)) return) does not eliminate
// inline string literals from external builds.
export function isVoiceGrowthBookEnabled(): boolean {
  return feature('VOICE_MODE')
    ? !getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_quartz_disabled', false)
    : false
}
```

**为什么使用正向模式：**
```typescript
// 错误模式 - 字符串可能泄漏
if (!feature('VOICE_MODE')) return false
return !getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_quartz_disabled', false)

// 正确模式 - 字符串被消除
return feature('VOICE_MODE')
  ? !getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_quartz_disabled', false)
  : false
```

### 4.5 USER_TYPE检查的内联要求

```typescript
// constants/prompts.ts:658
// DCE: inline the USER_TYPE check at each site — do NOT hoist to a const.
// Hoisting breaks constant folding because the comparison becomes a runtime
// variable reference, not a compile-time literal comparison.

// 错误 - 不会被DCE
const isAnt = process.env.USER_TYPE === 'ant'
if (isAnt && isUndercover()) { ... }

// 正确 - 会被DCE
if (process.env.USER_TYPE === 'ant' && isUndercover()) { ... }
```

---

## 5. GrowthBook集成

### 5.1 GrowthBook客户端初始化

```typescript
// services/analytics/growthbook.ts:490-617
const getGrowthBookClient = memoize(
  (): { client: GrowthBook; initialized: Promise<void> } | null => {
    if (!isGrowthBookEnabled()) {
      return null
    }

    const attributes = getUserAttributes()
    const clientKey = getGrowthBookClientKey()
    
    const baseUrl =
      process.env.USER_TYPE === 'ant'
        ? process.env.CLAUDE_CODE_GB_BASE_URL || 'https://api.anthropic.com/'
        : 'https://api.anthropic.com/'

    // Skip auth if trust hasn't been established yet
    const hasTrust =
      checkHasTrustDialogAccepted() ||
      getSessionTrustAccepted() ||
      getIsNonInteractiveSession()
    const authHeaders = hasTrust ? getAuthHeaders() : { headers: {}, error: 'trust not established' }

    const thisClient = new GrowthBook({
      apiHost: baseUrl,
      clientKey,
      attributes,
      remoteEval: true,  // 服务器端评估
      cacheKeyAttributes: ['id', 'organizationUUID'],
      ...(authHeaders.error ? {} : { apiHostRequestHeaders: authHeaders.headers }),
    })
    
    client = thisClient
    
    const initialized = thisClient
      .init({ timeout: 5000 })
      .then(async result => {
        const hadFeatures = await processRemoteEvalPayload(thisClient)
        if (hadFeatures) {
          syncRemoteEvalToDisk()
          refreshed.emit()
        }
      })
      .catch(error => { logError(toError(error)) })

    return { client: thisClient, initialized }
  },
)
```

### 5.2 用户属性传递

```typescript
// services/analytics/growthbook.ts:454-485
function getUserAttributes(): GrowthBookUserAttributes {
  const user = getUserForGrowthBook()

  // For ants, always try to include email from OAuth config
  let email = user.email
  if (!email && process.env.USER_TYPE === 'ant') {
    email = getGlobalConfig().oauthAccount?.emailAddress
  }

  const apiBaseUrlHost = getApiBaseUrlHost()

  const attributes = {
    id: user.deviceId,
    sessionId: user.sessionId,
    deviceID: user.deviceId,
    platform: user.platform,
    ...(apiBaseUrlHost && { apiBaseUrlHost }),
    ...(user.organizationUuid && { organizationUUID: user.accountUuid }),
    ...(user.userType && { userType: user.userType }),
    ...(user.subscriptionType && { subscriptionType: user.subscriptionType }),
    ...(user.rateLimitTier && { rateLimitTier: user.rateLimitTier }),
    ...(user.firstTokenTime && { firstTokenTime: user.firstTokenTime }),
    ...(email && { email }),
    ...(user.appVersion && { appVersion: user.appVersion }),
  }
  return attributes
}
```

**支持的属性：**
| 属性 | 用途 |
|------|------|
| `id` / `deviceID` | 设备唯一标识 |
| `sessionId` | 会话标识 |
| `platform` | 操作系统（win32/darwin/linux） |
| `organizationUUID` | 组织标识（企业用户） |
| `accountUUID` | 账户标识 |
| `userType` | 用户类型（ant/external） |
| `subscriptionType` | 订阅类型 |
| `rateLimitTier` | 速率限制等级 |
| `email` | 邮箱（用于定向投放） |
| `apiBaseUrlHost` | 代理服务器域名 |

### 5.3 远程评估处理

```typescript
// services/analytics/growthbook.ts:327-394
async function processRemoteEvalPayload(
  gbClient: GrowthBook,
): Promise<boolean> {
  // WORKAROUND: Transform remote eval response format
  // The API returns { "value": ... } but SDK expects { "defaultValue": ... }
  const payload = gbClient.getPayload()
  if (!payload?.features || Object.keys(payload.features).length === 0) {
    return false
  }

  // Clear before rebuild so features removed between refreshes don't leave stale entries
  experimentDataByFeature.clear()

  const transformedFeatures: Record<string, MalformedFeatureDefinition> = {}
  for (const [key, feature] of Object.entries(payload.features)) {
    const f = feature as MalformedFeatureDefinition
    if ('value' in f && !('defaultValue' in f)) {
      transformedFeatures[key] = {
        ...f,
        defaultValue: f.value,  // API格式修正
      }
    } else {
      transformedFeatures[key] = f
    }

    // Store experiment data for later logging
    if (f.source === 'experiment' && f.experimentResult) {
      const expResult = f.experimentResult as { variationId?: number }
      const exp = f.experiment as { key?: string } | undefined
      if (exp?.key && expResult.variationId !== undefined) {
        experimentDataByFeature.set(key, {
          experimentId: exp.key,
          variationId: expResult.variationId,
        })
      }
    }
  }
  
  // Re-set the payload with transformed features
  await gbClient.setPayload({ ...payload, features: transformedFeatures })

  // Cache evaluated values directly from remote eval response
  remoteEvalFeatureValues.clear()
  for (const [key, feature] of Object.entries(transformedFeatures)) {
    const v = 'value' in feature ? feature.value : feature.defaultValue
    if (v !== undefined) {
      remoteEvalFeatureValues.set(key, v)
    }
  }
  return true
}
```

### 5.4 获取Feature值

```typescript
// services/analytics/growthbook.ts:734-775
export function getFeatureValue_CACHED_MAY_BE_STALE<T>(
  feature: string,
  defaultValue: T,
): T {
  // 1. 检查环境变量覆盖（优先级最高）
  const overrides = getEnvOverrides()
  if (overrides && feature in overrides) {
    return overrides[feature] as T
  }
  
  // 2. 检查配置覆盖（ant-only）
  const configOverrides = getConfigOverrides()
  if (configOverrides && feature in configOverrides) {
    return configOverrides[feature] as T
  }

  if (!isGrowthBookEnabled()) {
    return defaultValue
  }

  // 3. 记录实验曝光
  if (experimentDataByFeature.has(feature)) {
    logExposureForFeature(feature)
  } else {
    pendingExposures.add(feature)
  }

  // 4. 内存缓存（最高优先级运行时）
  if (remoteEvalFeatureValues.has(feature)) {
    return remoteEvalFeatureValues.get(feature) as T
  }

  // 5. 磁盘缓存（启动时fallback）
  try {
    const cached = getGlobalConfig().cachedGrowthBookFeatures?.[feature]
    return cached !== undefined ? (cached as T) : defaultValue
  } catch {
    return defaultValue
  }
}
```

**优先级顺序：**
1. `CLAUDE_INTERNAL_FC_OVERRIDES` 环境变量（ant-only）
2. `growthBookOverrides` 本地配置（ant-only /config Gates tab）
3. 内存缓存（remoteEvalFeatureValues）
4. 磁盘缓存（cachedGrowthBookFeatures）
5. 默认值

### 5.5 周期性刷新

```typescript
// services/analytics/growthbook.ts:1012-1099
// Periodic refresh interval
const GROWTHBOOK_REFRESH_INTERVAL_MS =
  process.env.USER_TYPE !== 'ant'
    ? 6 * 60 * 60 * 1000  // 外部用户: 6小时
    : 20 * 60 * 1000      // Ant用户: 20分钟

export async function refreshGrowthBookFeatures(): Promise<void> {
  if (!isGrowthBookEnabled()) return

  const growthBookClient = await initializeGrowthBook()
  if (!growthBookClient) return

  await growthBookClient.refreshFeatures()

  // Rebuild remoteEvalFeatureValues from refreshed payload
  const hadFeatures = await processRemoteEvalPayload(growthBookClient)
  if (hadFeatures) {
    syncRemoteEvalToDisk()
    refreshed.emit()  // 通知订阅者
  }
}

export function setupPeriodicGrowthBookRefresh(): void {
  if (!isGrowthBookEnabled()) return

  if (refreshInterval) {
    clearInterval(refreshInterval)
  }

  refreshInterval = setInterval(() => {
    void refreshGrowthBookFeatures()
  }, GROWTHBOOK_REFRESH_INTERVAL_MS)
}
```

---

## 6. 用户配置

### 6.1 本地覆盖配置

```typescript
// utils/config.ts:451-453
export type GlobalConfig = {
  // Local GrowthBook overrides (ant-only, set via /config Gates tab).
  growthBookOverrides?: { [featureName: string]: unknown }
  cachedGrowthBookFeatures?: { [featureName: string]: unknown }
  cachedStatsigGates?: { [gateName: string]: boolean }
  ...
}
```

### 6.2 设置/清除覆盖

```typescript
// services/analytics/growthbook.ts:245-290
export function setGrowthBookConfigOverride(
  feature: string,
  value: unknown,
): void {
  if (process.env.USER_TYPE !== 'ant') return
  
  saveGlobalConfig(c => {
    const current = c.growthBookOverrides ?? {}
    if (value === undefined) {
      // 清除覆盖
      if (!(feature in current)) return c
      const { [feature]: _, ...rest } = current
      if (Object.keys(rest).length === 0) {
        const { growthBookOverrides: __, ...configWithout } = c
        return configWithout
      }
      return { ...c, growthBookOverrides: rest }
    }
    // 设置覆盖
    if (isEqual(current[feature], value)) return c
    return { ...c, growthBookOverrides: { ...current, [feature]: value } }
  })
  
  // 通知订阅者重新构建
  refreshed.emit()
}

export function clearGrowthBookConfigOverrides(): void {
  if (process.env.USER_TYPE !== 'ant') return
  
  saveGlobalConfig(c => {
    if (!c.growthBookOverrides || Object.keys(c.growthBookOverrides).length === 0) {
      return c
    }
    const { growthBookOverrides: _, ...rest } = c
    return rest
  })
  refreshed.emit()
}
```

### 6.3 环境变量覆盖

```typescript
// services/analytics/growthbook.ts:166-202
/**
 * Parse env var overrides for GrowthBook features.
 * Set CLAUDE_INTERNAL_FC_OVERRIDES to a JSON object mapping feature keys to values
 * to bypass remote eval and disk cache. Only active when USER_TYPE is 'ant'.
 *
 * Example: CLAUDE_INTERNAL_FC_OVERRIDES='{"my_feature": true, "my_config": {"key": "val"}}'
 */
let envOverrides: Record<string, unknown> | null = null
let envOverridesParsed = false

function getEnvOverrides(): Record<string, unknown> | null {
  if (!envOverridesParsed) {
    envOverridesParsed = true
    if (process.env.USER_TYPE === 'ant') {
      const raw = process.env.CLAUDE_INTERNAL_FC_OVERRIDES
      if (raw) {
        try {
          envOverrides = JSON.parse(raw) as Record<string, unknown>
          logForDebugging(
            `GrowthBook: Using env var overrides for ${Object.keys(envOverrides!).length} features`,
          )
        } catch {
          logError(new Error(`GrowthBook: Failed to parse CLAUDE_INTERNAL_FC_OVERRIDES: ${raw}`))
        }
      }
    }
  }
  return envOverrides
}

export function hasGrowthBookEnvOverride(feature: string): boolean {
  const overrides = getEnvOverrides()
  return overrides !== null && feature in overrides
}
```

### 6.4 刷新订阅机制

```typescript
// services/analytics/growthbook.ts:106-157
type GrowthBookRefreshListener = () => void | Promise<void>
const refreshed = createSignal()

export function onGrowthBookRefresh(
  listener: GrowthBookRefreshListener,
): () => void {
  let subscribed = true
  const unsubscribe = refreshed.subscribe(() => callSafe(listener))
  
  // 如果已有缓存数据，立即触发一次
  if (remoteEvalFeatureValues.size > 0) {
    queueMicrotask(() => {
      if (subscribed && remoteEvalFeatureValues.size > 0) {
        callSafe(listener)
      }
    })
  }
  
  return () => {
    subscribed = false
    unsubscribe()
  }
}
```

**使用场景：**
```typescript
// 典型使用：启动时订阅，feature变更时重新构建
useEffect(() => {
  const unsubscribe = onGrowthBookRefresh(() => {
    // feature值可能已更新，需要重新计算
    setSomeState(getFeatureValue_CACHED_MAY_BE_STALE('some_feature', defaultValue))
  })
  return unsubscribe
}, [])
```

---

## 7. 从零实现

### 7.1 编译时Feature Flag实现

**Step 1: 定义构建配置**

```bash
# 构建脚本（build.sh）
#!/bin/bash

# 外部构建
bun build ./src/main.tsx \
  --outfile ./dist/claude-code-external.js \
  --define USER_TYPE='"external"' \
  --define feature.VOICE_MODE=false \
  --define feature.BUDDY=false \
  --define feature.KAIROS=false \
  --define feature.BRIDGE_MODE=false \
  --define feature.PROACTIVE=false \
  --define feature.DAEMON=false \
  --define feature.ULTRAPLAN=false \
  --define feature.FORK_SUBAGENT=false

# Ant构建
bun build ./src/main.tsx \
  --outfile ./dist/claude-code-ant.js \
  --define USER_TYPE='"ant"' \
  --define feature.VOICE_MODE=true \
  --define feature.BUDDY=true \
  --define feature.KAIROS=true \
  --define feature.BRIDGE_MODE=true \
  --define feature.PROACTIVE=true \
  --define feature.DAEMON=true \
  --define feature.ULTRAPLAN=true \
  --define feature.FORK_SUBAGENT=true
```

**Step 2: 实现条件导入**

```typescript
// src/commands.ts
import { feature } from 'bun:bundle'

// 条件导入模块
/* eslint-disable @typescript-eslint/no-require-imports */
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null

const buddyCommand = feature('BUDDY')
  ? require('./commands/buddy/index.js').default
  : null
/* eslint-enable @typescript-eslint/no-require-imports */

// 注册命令（null会被跳过）
export function getCommands(): Command[] {
  return [
    voiceCommand,
    buddyCommand,
    // ...其他命令
  ].filter(Boolean) as Command[]
}
```

**Step 3: 条件执行代码**

```typescript
// src/main.tsx
import { feature } from 'bun:bundle'

async function main() {
  // DCE: 外部构建中此块被完全消除
  if (feature('VOICE_MODE') && args[0] === 'voice') {
    const { startVoiceMode } = await import('./voice/index.js')
    await startVoiceMode()
    return
  }

  // DCE: 外部构建中此块被完全消除
  if (feature('BUDDY') && args[0] === 'buddy') {
    const { hatchBuddy } = await import('./buddy/index.js')
    await hatchBuddy()
    return
  }
  
  // 常规逻辑...
}
```

### 7.2 运行时Feature Flag实现

**Step 1: 安装GrowthBook SDK**

```bash
npm install @growthbook/growthbook
```

**Step 2: 创建GrowthBook服务**

```typescript
// src/services/analytics/growthbook.ts
import { GrowthBook } from '@growthbook/growthbook'
import { memoize } from 'lodash-es'
import { getGlobalConfig, saveGlobalConfig } from '../utils/config.js'

// GrowthBook客户端
let client: GrowthBook | null = null
const remoteEvalFeatureValues = new Map<string, unknown>()

// 用户属性
type UserAttributes = {
  id: string
  sessionId: string
  deviceID: string
  platform: 'win32' | 'darwin' | 'linux'
  email?: string
  userType?: string
}

function getUserAttributes(): UserAttributes {
  return {
    id: getDeviceId(),
    sessionId: getSessionId(),
    deviceID: getDeviceId(),
    platform: process.platform,
    email: getEmail(),
    userType: process.env.USER_TYPE,
  }
}

// 初始化客户端
export const initializeGrowthBook = memoize(
  async (): Promise<GrowthBook | null> => {
    const clientKey = getGrowthBookClientKey()
    const attributes = getUserAttributes()
    
    const gbClient = new GrowthBook({
      apiHost: 'https://api.anthropic.com/',
      clientKey,
      attributes,
      remoteEval: true,  // 服务器端评估
      cacheKeyAttributes: ['id'],  // 缓存键属性
    })
    
    client = gbClient
    
    await gbClient.init({ timeout: 5000 })
    
    // 处理远程评估结果
    const payload = gbClient.getPayload()
    if (payload?.features) {
      remoteEvalFeatureValues.clear()
      for (const [key, feature] of Object.entries(payload.features)) {
        const f = feature as { value?: unknown; defaultValue?: unknown }
        const v = f.value ?? f.defaultValue
        if (v !== undefined) {
          remoteEvalFeatureValues.set(key, v)
        }
      }
      // 同步到磁盘
      saveGlobalConfig(c => ({
        ...c,
        cachedGrowthBookFeatures: Object.fromEntries(remoteEvalFeatureValues),
      }))
    }
    
    return gbClient
  }
)

// 获取feature值（非阻塞）
export function getFeatureValue_CACHED_MAY_BE_STALE<T>(
  feature: string,
  defaultValue: T,
): T {
  // 检查内存缓存
  if (remoteEvalFeatureValues.has(feature)) {
    return remoteEvalFeatureValues.get(feature) as T
  }
  
  // 检查磁盘缓存
  try {
    const cached = getGlobalConfig().cachedGrowthBookFeatures?.[feature]
    return cached !== undefined ? (cached as T) : defaultValue
  } catch {
    return defaultValue
  }
}

// 周期性刷新
const REFRESH_INTERVAL_MS = 6 * 60 * 60 * 1000  // 6小时

export function setupPeriodicGrowthBookRefresh(): void {
  setInterval(async () => {
    if (!client) return
    await client.refreshFeatures()
    
    const payload = client.getPayload()
    if (payload?.features) {
      remoteEvalFeatureValues.clear()
      for (const [key, feature] of Object.entries(payload.features)) {
        const f = feature as { value?: unknown; defaultValue?: unknown }
        const v = f.value ?? f.defaultValue
        if (v !== undefined) {
          remoteEvalFeatureValues.set(key, v)
        }
      }
      saveGlobalConfig(c => ({
        ...c,
        cachedGrowthBookFeatures: Object.fromEntries(remoteEvalFeatureValues),
      }))
    }
  }, REFRESH_INTERVAL_MS)
}
```

**Step 3: 在业务代码中使用**

```typescript
// src/utils/agentSwarmsEnabled.ts
import { getFeatureValue_CACHED_MAY_BE_STALE } from '../services/analytics/growthbook.js'

export function isAgentSwarmsEnabled(): boolean {
  // Ant用户始终启用
  if (process.env.USER_TYPE === 'ant') {
    return true
  }
  
  // 外部用户需要opt-in + GrowthBook gate
  if (!isEnvTruthy(process.env.CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS)) {
    return false
  }
  
  // Killswitch检查
  return getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_flint', true)
}
```

**Step 4: 实现本地覆盖**

```typescript
// 添加到 growthbook.ts
export function setGrowthBookConfigOverride(
  feature: string,
  value: unknown,
): void {
  saveGlobalConfig(c => {
    const current = c.growthBookOverrides ?? {}
    if (value === undefined) {
      const { [feature]: _, ...rest } = current
      return { ...c, growthBookOverrides: rest }
    }
    return { ...c, growthBookOverrides: { ...current, [feature]: value } }
  })
}

// 修改 getFeatureValue_CACHED_MAY_BE_STALE 添加覆盖检查
export function getFeatureValue_CACHED_MAY_BE_STALE<T>(
  feature: string,
  defaultValue: T,
): T {
  // 优先检查本地覆盖
  const config = getGlobalConfig()
  if (config.growthBookOverrides && feature in config.growthBookOverrides) {
    return config.growthBookOverrides[feature] as T
  }
  
  // ...其余逻辑不变
}
```

### 7.3 完整示例：语音模式Feature

```typescript
// src/voice/voiceModeEnabled.ts
import { feature } from 'bun:bundle'
import { getFeatureValue_CACHED_MAY_BE_STALE } from '../services/analytics/growthbook.js'
import { hasVoiceAuth } from '../utils/auth.js'

/**
 * 编译时开关 - 外部构建中此函数返回false
 * 运行时GrowthBook检查代码被DCE消除
 */
export function isVoiceGrowthBookEnabled(): boolean {
  // 正向三元模式 - 确保DCE消除字符串
  return feature('VOICE_MODE')
    ? !getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_quartz_disabled', false)
    : false
}

/**
 * 运行时完整检查 - auth + GrowthBook
 */
export function isVoiceModeEnabled(): boolean {
  // 编译时：外部构建中直接返回false（DCE）
  if (!feature('VOICE_MODE')) {
    return false
  }
  
  // 运行时：检查auth和GrowthBook
  return hasVoiceAuth() && isVoiceGrowthBookEnabled()
}

// src/commands/voice/index.ts - 仅在feature启用时被打包
export default {
  type: 'prompt',
  name: 'voice',
  description: 'Start voice mode',
  async getPromptForCommand(args, context) {
    if (!isVoiceModeEnabled()) {
      return 'Voice mode is not enabled for your account.'
    }
    // 语音模式逻辑...
  },
}

// src/commands.ts - 条件导入
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null
```

---

## 8. 最佳实践

### 8.1 编译时Feature最佳实践

1. **使用正向三元模式**
   ```typescript
   // 好
   return feature('X') ? someCode() : false
   
   // 差 - 字符串可能泄漏
   if (!feature('X')) return false
   return someCode()
   ```

2. **内联USER_TYPE检查**
   ```typescript
   // 好 - DCE有效
   if (process.env.USER_TYPE === 'ant' && someCondition()) { }
   
   // 差 - DCE失效
   const isAnt = process.env.USER_TYPE === 'ant'
   if (isAnt && someCondition()) { }
   ```

3. **注意DCE复杂度悬崖**
   ```typescript
   // 好 - 顶层const重绑定
   const helper = importedHelper_DEPRECATED
   
   // 差 - import别名消耗复杂度预算
   import { helper as importedHelper_DEPRECATED } from './module.js'
   ```

### 8.2 运行时Feature最佳实践

1. **使用CACHED_MAY_BE_STALE版本**
   ```typescript
   // 好 - 非阻塞，启动友好
   const value = getFeatureValue_CACHED_MAY_BE_STALE('feature', defaultValue)
   
   // 差 - 阻塞启动
   const value = await getFeatureValue_DEPRECATED('feature', defaultValue)
   ```

2. **订阅刷新事件**
   ```typescript
   // 对长期对象需要订阅刷新
   useEffect(() => {
     const unsubscribe = onGrowthBookRefresh(() => {
       rebuildConfig()
     })
     return unsubscribe
   }, [])
   ```

3. **组合编译时和运行时检查**
   ```typescript
   export function isFeatureEnabled(): boolean {
     // 编译时开关（DCE）
     if (!feature('FEATURE_NAME')) {
       return false
     }
     // 运行时动态控制
     return getFeatureValue_CACHED_MAY_BE_STALE('feature_gate', false)
   }
   ```

---

## 9. 关键文件索引

| 文件路径 | 功能 |
|---------|------|
| `services/analytics/growthbook.ts` | GrowthBook客户端、feature值获取 |
| `constants/betas.ts` | Beta headers定义，条件编译beta |
| `utils/betas.ts` | Beta headers合并逻辑 |
| `voice/voiceModeEnabled.ts` | 语音模式feature检查示例 |
| `utils/agentSwarmsEnabled.ts` | Agent teams feature检查示例 |
| `utils/worktreeModeEnabled.ts` | Worktree模式feature（已无条件启用） |
| `commands.ts:60-123` | 编译时条件导入示例 |
| `constants/prompts.ts:64-98` | 条件模块导入示例 |
| `entrypoints/cli.tsx:19-71` | 条件执行块示例 |
| `tools/BashTool/bashPermissions.ts:81-89` | DCE复杂度悬崖说明 |
| `utils/config.ts:451-453` | GlobalConfig类型定义 |
| `constants/keys.ts:5-11` | GrowthBook SDK Key选择 |

---

*Feature Flag系统分析完成*