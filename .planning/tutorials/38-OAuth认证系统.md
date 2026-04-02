# OAuth认证系统 - 逐行细粒度分析

**分析日期:** 2026-04-02

## 目录

1. [OAuth流程概述](#1-oauth流程概述)
2. [PKCE实现](#2-pkce实现)
3. [services/oauth/分析](#3-servicesoauth分析)
4. [Token刷新机制](#4-token刷新机制)
5. [多认证源优先级](#5-多认证源优先级)
6. [用户使用指南](#6-用户使用指南)
7. [从零实现](#7-从零实现)

---

## 1. OAuth流程概述

### 1.1 整体架构

Claude Code CLI 实现了完整的 OAuth 2.0 Authorization Code Flow with PKCE (Proof Key for Code Exchange)。

**核心文件:**
- `services/oauth/index.ts` - OAuthService 主服务
- `services/oauth/client.ts` - OAuth客户端，处理token交换
- `services/oauth/crypto.ts` - PKCE加密工具
- `services/oauth/auth-code-listener.ts` - 本地回调监听器
- `services/oauth/getOauthProfile.ts` - 用户信息获取

**流程图:**

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   CLI启动    │ --> │ 生成PKCE参数 │ --> │ 启动本地监听 │
└──────────────┘     └──────────────┘     └──────────────┘
                                               │
                                               v
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   用户授权   │ --> │ 回调接收code │ --> │ 交换获取token│
└──────────────┘     └──────────────┘     └──────────────┘
                                               │
                                               v
                     ┌──────────────┐     ┌──────────────┐
                     │  安全存储    │ --> │  后续请求使用│
                     └──────────────┘     └──────────────┘
```

### 1.2 双重认证路径

系统支持两种认证方式:

**自动流程 (Automatic Flow):**
```typescript
// services/oauth/index.ts:51-86
async startOAuthFlow(authURLHandler, options) {
  // 1. 创建本地回调监听器
  this.authCodeListener = new AuthCodeListener()
  this.port = await this.authCodeListener.start()

  // 2. 生成PKCE参数
  const codeChallenge = crypto.generateCodeChallenge(this.codeVerifier)
  const state = crypto.generateState()

  // 3. 构建认证URL
  const automaticFlowUrl = client.buildAuthUrl({
    codeChallenge, state, port: this.port,
    isManual: false, // 自动模式
    loginWithClaudeAi: options?.loginWithClaudeAi,
    ...
  })

  // 4. 打开浏览器进行授权
  await openBrowser(automaticFlowUrl)

  // 5. 等待回调并交换token
  const tokenResponse = await client.exchangeCodeForTokens(...)
}
```

**手动流程 (Manual Flow):**
用于无浏览器环境(如SSH远程连接):
```typescript
// services/oauth/index.ts:69-70
const manualFlowUrl = client.buildAuthUrl({
  ...opts,
  isManual: true // 手动模式，使用固定回调URL
})

// 用户手动复制认证码
handleManualAuthCodeInput({ authorizationCode, state })
```

### 1.3 OAuth配置 (constants/oauth.ts)

**配置结构:**
```typescript
// constants/oauth.ts:60-81
type OauthConfig = {
  BASE_API_URL: string          // API基础URL
  CONSOLE_AUTHORIZE_URL: string // Console授权端点
  CLAUDE_AI_AUTHORIZE_URL: string // claude.ai授权端点
  CLAUDE_AI_ORIGIN: string      // claude.ai域名
  TOKEN_URL: string             // Token交换端点
  API_KEY_URL: string           // API密钥创建端点
  ROLES_URL: string             // 用户角色获取端点
  CONSOLE_SUCCESS_URL: string   // Console成功页面
  CLAUDEAI_SUCCESS_URL: string  // claude.ai成功页面
  MANUAL_REDIRECT_URL: string   // 手动流程回调URL
  CLIENT_ID: string             // OAuth客户端ID
  OAUTH_FILE_SUFFIX: string     // OAuth文件后缀
  MCP_PROXY_URL: string         // MCP代理URL
  MCP_PROXY_PATH: string        // MCP代理路径
}
```

**生产环境配置:**
```typescript
// constants/oauth.ts:84-104
const PROD_OAUTH_CONFIG = {
  BASE_API_URL: 'https://api.anthropic.com',
  CONSOLE_AUTHORIZE_URL: 'https://platform.claude.com/oauth/authorize',
  CLAUDE_AI_AUTHORIZE_URL: 'https://claude.com/cai/oauth/authorize',
  CLAUDE_AI_ORIGIN: 'https://claude.ai',
  TOKEN_URL: 'https://platform.claude.com/v1/oauth/token',
  CLIENT_ID: '9d1c250a-e61b-44d9-88ed-5944d1962f5e',
  ...
}
```

**环境切换机制:**
```typescript
// constants/oauth.ts:6-16
function getOauthConfigType(): OauthConfigType {
  if (process.env.USER_TYPE === 'ant') {
    // Anthropic内部员工可以使用测试环境
    if (isEnvTruthy(process.env.USE_LOCAL_OAUTH)) {
      return 'local'
    }
    if (isEnvTruthy(process.env.USE_STAGING_OAUTH)) {
      return 'staging'
    }
  }
  return 'prod' // 默认生产环境
}
```

---

## 2. PKCE实现

### 2.1 PKCE概述

PKCE (Proof Key for Code Exchange) 是 OAuth 2.0 的安全扩展，防止授权码拦截攻击。

**核心原理:**
1. 客户端生成随机 `code_verifier`
2. 使用SHA256计算 `code_challenge`
3. 授权请求携带 `code_challenge`
4. Token交换携带 `code_verifier`
5. 服务端验证匹配关系

### 2.2 crypto.ts 逐行分析

**文件位置:** `services/oauth/crypto.ts`

```typescript
// services/oauth/crypto.ts:1-24
import { createHash, randomBytes } from 'crypto'

// Base64 URL安全编码
function base64URLEncode(buffer: Buffer): string {
  return buffer
    .toString('base64')
    .replace(/\+/g, '-')  // + -> -
    .replace(/\//g, '_')  // / -> _
    .replace(/=/g, '')    // 移除填充
}
```

**关键技术点:**

1. **Base64 URL编码转换:**
   - 标准 Base64 包含 `+` 和 `/`，不适合URL传输
   - 替换为 `-` 和 `_` 符合 RFC 7636 规范
   - 移除 `=` 填充符，减少URL长度

2. **Code Verifier生成 (32字节随机):**
```typescript
// services/oauth/crypto.ts:11-13
export function generateCodeVerifier(): string {
  return base64URLEncode(randomBytes(32))
}
```
   - 使用 Node.js `crypto.randomBytes(32)` 生成256位随机值
   - 编码后约43字符，符合 RFC 7636 最小长度要求(43-128字符)

3. **Code Challenge计算 (SHA256):**
```typescript
// services/oauth/crypto.ts:15-18
export function generateCodeChallenge(verifier: string): string {
  const hash = createHash('sha256')
  hash.update(verifier)
  return base64URLEncode(hash.digest())
}
```
   - 使用 SHA256 哈希算法
   - `S256` 方法比 `plain` 更安全(不暴露verifier)

4. **State参数生成:**
```typescript
// services/oauth/crypto.ts:21-23
export function generateState(): string {
  return base64URLEncode(randomBytes(32))
}
```
   - State参数防止 CSRF 攻击
   - 回调时验证state匹配

### 2.3 PKCE流程集成

**授权URL构建:**
```typescript
// services/oauth/client.ts:46-105
export function buildAuthUrl({
  codeChallenge,
  state,
  port,
  isManual,
  ...
}): string {
  const authUrl = new URL(authUrlBase)

  // OAuth标准参数
  authUrl.searchParams.append('client_id', getOauthConfig().CLIENT_ID)
  authUrl.searchParams.append('response_type', 'code')
  authUrl.searchParams.append('redirect_uri', isManual
    ? getOauthConfig().MANUAL_REDIRECT_URL
    : `http://localhost:${port}/callback`)

  // PKCE参数
  authUrl.searchParams.append('code_challenge', codeChallenge)
  authUrl.searchParams.append('code_challenge_method', 'S256')

  // CSRF防护
  authUrl.searchParams.append('state', state)

  // Scope请求
  authUrl.searchParams.append('scope', scopesToUse.join(' '))

  return authUrl.toString()
}
```

**Token交换验证:**
```typescript
// services/oauth/client.ts:107-144
export async function exchangeCodeForTokens(
  authorizationCode: string,
  state: string,
  codeVerifier: string,  // PKCE验证
  port: number,
  useManualRedirect: boolean,
  expiresIn?: number,
): Promise<OAuthTokenExchangeResponse> {
  const requestBody: Record<string, string | number> = {
    grant_type: 'authorization_code',
    code: authorizationCode,
    redirect_uri: useManualRedirect
      ? getOauthConfig().MANUAL_REDIRECT_URL
      : `http://localhost:${port}/callback`,
    client_id: getOauthConfig().CLIENT_ID,
    code_verifier: codeVerifier,  // 关键：发送verifier供验证
    state,
  }

  const response = await axios.post(getOauthConfig().TOKEN_URL, requestBody, {
    headers: { 'Content-Type': 'application/json' },
    timeout: 15000,
  })
  ...
}
```

---

## 3. services/oauth/ 分析

### 3.1 OAuthService 类 (index.ts)

**类结构:**
```typescript
// services/oauth/index.ts:21-30
export class OAuthService {
  private codeVerifier: string           // PKCE验证器
  private authCodeListener: AuthCodeListener | null = null  // 回调监听
  private port: number | null = null     // 本地端口
  private manualAuthCodeResolver: ((authorizationCode: string) => void) | null = null

  constructor() {
    this.codeVerifier = crypto.generateCodeVerifier()  // 初始化PKCE
  }
}
```

**核心方法 startOAuthFlow:**
```typescript
// services/oauth/index.ts:32-132
async startOAuthFlow(
  authURLHandler: (url: string, automaticUrl?: string) => Promise<void>,
  options?: {
    loginWithClaudeAi?: boolean    // 使用claude.ai登录
    inferenceOnly?: boolean        // 仅推理权限
    expiresIn?: number             // Token有效期
    orgUUID?: string               // 组织UUID
    loginHint?: string             // 预填充邮箱
    loginMethod?: string           // 登录方式(sso/magic_link/google)
    skipBrowserOpen?: boolean      // 跳过浏览器打开
  },
): Promise<OAuthTokens> {
  // Step 1: 启动本地回调服务器
  this.authCodeListener = new AuthCodeListener()
  this.port = await this.authCodeListener.start()

  // Step 2: 生成PKCE和安全参数
  const codeChallenge = crypto.generateCodeChallenge(this.codeVerifier)
  const state = crypto.generateState()

  // Step 3: 构建双重URL
  const manualFlowUrl = client.buildAuthUrl({ ...opts, isManual: true })
  const automaticFlowUrl = client.buildAuthUrl({ ...opts, isManual: false })

  // Step 4: 等待授权码(自动或手动)
  const authorizationCode = await this.waitForAuthorizationCode(state, async () => {
    if (options?.skipBrowserOpen) {
      // SDK控制协议：交由调用方处理
      await authURLHandler(manualFlowUrl, automaticFlowUrl)
    } else {
      await authURLHandler(manualFlowUrl)  // 显示手动选项
      await openBrowser(automaticFlowUrl)  // 尝试自动流程
    }
  })

  // Step 5: 判断是否自动流程
  const isAutomaticFlow = this.authCodeListener?.hasPendingResponse() ?? false

  // Step 6: 交换Token
  const tokenResponse = await client.exchangeCodeForTokens(
    authorizationCode,
    state,
    this.codeVerifier,  // PKCE验证
    this.port!,
    !isAutomaticFlow,   // 手动模式标志
    options?.expiresIn,
  )

  // Step 7: 获取用户信息
  const profileInfo = await client.fetchProfileInfo(tokenResponse.access_token)

  // Step 8: 自动流程成功重定向
  if (isAutomaticFlow) {
    const scopes = client.parseScopes(tokenResponse.scope)
    this.authCodeListener?.handleSuccessRedirect(scopes)
  }

  // Step 9: 格式化返回结果
  return this.formatTokens(tokenResponse, profileInfo.subscriptionType, ...)
}
```

**Token格式化:**
```typescript
// services/oauth/index.ts:169-191
private formatTokens(
  response: OAuthTokenExchangeResponse,
  subscriptionType: SubscriptionType | null,
  rateLimitTier: RateLimitTier | null,
  profile?: OAuthProfileResponse,
): OAuthTokens {
  return {
    accessToken: response.access_token,
    refreshToken: response.refresh_token,
    expiresAt: Date.now() + response.expires_in * 1000,  // 计算过期时间
    scopes: client.parseScopes(response.scope),
    subscriptionType,      // 订阅类型(max/pro/team/enterprise)
    rateLimitTier,         // 速率限制级别
    profile,
    tokenAccount: response.account ? {
      uuid: response.account.uuid,
      emailAddress: response.account.email_address,
      organizationUuid: response.organization?.uuid,
    } : undefined,
  }
}
```

### 3.2 AuthCodeListener 类 (auth-code-listener.ts)

**用途:** 创建临时本地HTTP服务器监听OAuth回调

**类结构:**
```typescript
// services/oauth/auth-code-listener.ts:18-30
export class AuthCodeListener {
  private localServer: Server               // HTTP服务器
  private port: number = 0                  // 监听端口
  private promiseResolver: ((code: string) => void) | null = null
  private promiseRejecter: ((error: Error) => void) | null = null
  private expectedState: string | null = null  // 期望的state值
  private pendingResponse: ServerResponse | null = null  // 待处理响应
  private callbackPath: string               // 回调路径

  constructor(callbackPath: string = '/callback') {
    this.localServer = createServer()
    this.callbackPath = callbackPath
  }
}
```

**端口启动(OS分配):**
```typescript
// services/oauth/auth-code-listener.ts:37-52
async start(port?: number): Promise<number> {
  return new Promise((resolve, reject) => {
    this.localServer.once('error', err => {
      reject(new Error(`Failed to start OAuth callback server: ${err.message}`))
    })

    // port=0 让OS分配可用端口，避免端口冲突
    this.localServer.listen(port ?? 0, 'localhost', () => {
      const address = this.localServer.address() as AddressInfo
      this.port = address.port
      resolve(this.port)
    })
  })
}
```

**回调处理:**
```typescript
// services/oauth/auth-code-listener.ts:134-150
private handleRedirect(req: IncomingMessage, res: ServerResponse): void {
  const parsedUrl = new URL(
    req.url || '',
    `http://${req.headers.host || 'localhost'}`,
  )

  // 验证回调路径
  if (parsedUrl.pathname !== this.callbackPath) {
    res.writeHead(404)
    res.end()
    return
  }

  // 提取授权码和state
  const authCode = parsedUrl.searchParams.get('code') ?? undefined
  const state = parsedUrl.searchParams.get('state') ?? undefined

  this.validateAndRespond(authCode, state, res)
}
```

**安全验证:**
```typescript
// services/oauth/auth-code-listener.ts:152-175
private validateAndRespond(
  authCode: string | undefined,
  state: string | undefined,
  res: ServerResponse,
): void {
  // 验证授权码存在
  if (!authCode) {
    res.writeHead(400)
    res.end('Authorization code not found')
    this.reject(new Error('No authorization code received'))
    return
  }

  // 验证state匹配(CSRF防护)
  if (state !== this.expectedState) {
    res.writeHead(400)
    res.end('Invalid state parameter')
    this.reject(new Error('Invalid state parameter'))
    return
  }

  // 存储响应对象用于后续重定向
  this.pendingResponse = res

  this.resolve(authCode)
}
```

**成功重定向:**
```typescript
// services/oauth/auth-code-listener.ts:80-105
handleSuccessRedirect(scopes: string[], customHandler?): void {
  if (!this.pendingResponse) return

  // 根据scope选择成功页面
  const successUrl = shouldUseClaudeAIAuth(scopes)
    ? getOauthConfig().CLAUDEAI_SUCCESS_URL
    : getOauthConfig().CONSOLE_SUCCESS_URL

  // 302重定向到成功页面
  this.pendingResponse.writeHead(302, { Location: successUrl })
  this.pendingResponse.end()
  this.pendingResponse = null

  logEvent('tengu_oauth_automatic_redirect', {})
}
```

### 3.3 OAuth客户端 (client.ts)

**核心功能:** Token交换、刷新、用户信息获取

**Scope判断:**
```typescript
// services/oauth/client.ts:38-44
export function shouldUseClaudeAIAuth(scopes: string[] | undefined): boolean {
  return Boolean(scopes?.includes(CLAUDE_AI_INFERENCE_SCOPE))
}

export function parseScopes(scopeString?: string): string[] {
  return scopeString?.split(' ').filter(Boolean) ?? []
}
```

**Token刷新 (核心):**
```typescript
// services/oauth/client.ts:146-274
export async function refreshOAuthToken(
  refreshToken: string,
  { scopes: requestedScopes }: { scopes?: string[] } = {},
): Promise<OAuthTokens> {
  const requestBody = {
    grant_type: 'refresh_token',
    refresh_token: refreshToken,
    client_id: getOauthConfig().CLIENT_ID,
    // 默认使用Claude AI scope集，允许scope扩展
    scope: (requestedScopes?.length
      ? requestedScopes
      : CLAUDE_AI_OAUTH_SCOPES
    ).join(' '),
  }

  const response = await axios.post(getOauthConfig().TOKEN_URL, requestBody, {
    headers: { 'Content-Type': 'application/json' },
    timeout: 15000,
  })

  const data = response.data as OAuthTokenExchangeResponse
  const {
    access_token: accessToken,
    refresh_token: newRefreshToken = refreshToken,  // 可能返回新refresh_token
    expires_in: expiresIn,
  } = data

  const expiresAt = Date.now() + expiresIn * 1000
  const scopes = parseScopes(data.scope)

  // 智能缓存策略：已有完整profile信息时跳过API调用
  const config = getGlobalConfig()
  const existing = getClaudeAIOAuthTokens()
  const haveProfileAlready =
    config.oauthAccount?.billingType !== undefined &&
    config.oauthAccount?.accountCreatedAt !== undefined &&
    config.oauthAccount?.subscriptionCreatedAt !== undefined &&
    existing?.subscriptionType != null &&
    existing?.rateLimitTier != null

  const profileInfo = haveProfileAlready
    ? null
    : await fetchProfileInfo(accessToken)

  // 更新OAuth账户信息
  if (profileInfo && config.oauthAccount) {
    const updates: Partial<AccountInfo> = {}
    if (profileInfo.displayName !== undefined) {
      updates.displayName = profileInfo.displayName
    }
    if (typeof profileInfo.hasExtraUsageEnabled === 'boolean') {
      updates.hasExtraUsageEnabled = profileInfo.hasExtraUsageEnabled
    }
    // ... 更多字段更新
    saveGlobalConfig(current => ({
      ...current,
      oauthAccount: current.oauthAccount
        ? { ...current.oauthAccount, ...updates }
        : current.oauthAccount,
    }))
  }

  return {
    accessToken,
    refreshToken: newRefreshToken,
    expiresAt,
    scopes,
    subscriptionType: profileInfo?.subscriptionType ?? existing?.subscriptionType ?? null,
    rateLimitTier: profileInfo?.rateLimitTier ?? existing?.rateLimitTier ?? null,
    profile: profileInfo?.rawProfile,
    tokenAccount: data.account ? {
      uuid: data.account.uuid,
      emailAddress: data.account.email_address,
      organizationUuid: data.organization?.uuid,
    } : undefined,
  }
}
```

**用户Profile获取:**
```typescript
// services/oauth/client.ts:355-420
export async function fetchProfileInfo(accessToken: string): Promise<{
  subscriptionType: SubscriptionType | null
  displayName?: string
  rateLimitTier: RateLimitTier | null
  hasExtraUsageEnabled: boolean | null
  billingType: BillingType | null
  accountCreatedAt?: string
  subscriptionCreatedAt?: string
  rawProfile?: OAuthProfileResponse
}> {
  const profile = await getOauthProfileFromOauthToken(accessToken)
  const orgType = profile?.organization?.organization_type

  // 根据组织类型确定订阅类型
  let subscriptionType: SubscriptionType | null = null
  switch (orgType) {
    case 'claude_max':
      subscriptionType = 'max'
      break
    case 'claude_pro':
      subscriptionType = 'pro'
      break
    case 'claude_enterprise':
      subscriptionType = 'enterprise'
      break
    case 'claude_team':
      subscriptionType = 'team'
      break
    default:
      subscriptionType = null
      break
  }

  return {
    subscriptionType,
    rateLimitTier: profile?.organization?.rate_limit_tier ?? null,
    hasExtraUsageEnabled: profile?.organization?.has_extra_usage_enabled ?? null,
    billingType: profile?.organization?.billing_type ?? null,
    displayName: profile?.account?.display_name,
    accountCreatedAt: profile?.account?.created_at,
    subscriptionCreatedAt: profile?.organization?.subscription_created_at,
    rawProfile: profile,
  }
}
```

### 3.4 getOauthProfile.ts

**API请求封装:**
```typescript
// services/oauth/getOauthProfile.ts:7-35
export async function getOauthProfileFromApiKey(): Promise<OAuthProfileResponse | undefined> {
  const config = getGlobalConfig()
  const accountUuid = config.oauthAccount?.accountUuid
  const apiKey = getAnthropicApiKey()

  if (!accountUuid || !apiKey) return

  const endpoint = `${getOauthConfig().BASE_API_URL}/api/claude_cli_profile`
  const response = await axios.get<OAuthProfileResponse>(endpoint, {
    headers: {
      'x-api-key': apiKey,
      'anthropic-beta': OAUTH_BETA_HEADER,
    },
    params: { account_uuid: accountUuid },
    timeout: 10000,
  })
  return response.data
}

export async function getOauthProfileFromOauthToken(
  accessToken: string,
): Promise<OAuthProfileResponse | undefined> {
  const endpoint = `${getOauthConfig().BASE_API_URL}/api/oauth/profile`
  const response = await axios.get<OAuthProfileResponse>(endpoint, {
    headers: {
      Authorization: `Bearer ${accessToken}`,
      'Content-Type': 'application/json',
    },
    timeout: 10000,
  })
  return response.data
}
```

---

## 4. Token刷新机制

### 4.1 刷新触发条件

**过期检测 (utils/auth.ts引用):**
```typescript
// services/oauth/client.ts:344-353
export function isOAuthTokenExpired(expiresAt: number | null): boolean {
  if (expiresAt === null) {
    return false  // 无过期时间的token(环境变量token)不触发刷新
  }

  // 5分钟缓冲时间，提前刷新避免临界点失败
  const bufferTime = 5 * 60 * 1000
  const now = Date.now()
  const expiresWithBuffer = now + bufferTime
  return expiresWithBuffer >= expiresAt
}
```

**缓冲机制原理:**
- Token有效期通常为1小时
- 提前5分钟刷新确保请求时token有效
- 避免"刚好过期"的临界情况

### 4.2 刷新流程 (utils/auth.ts)

**主刷新函数:**
```typescript
// utils/auth.ts:1427-1562
export function checkAndRefreshOAuthTokenIfNeeded(
  retryCount = 0,
  force = false,
): Promise<boolean> {
  // 并发请求去重：多个connector同时401时只刷新一次
  if (retryCount === 0 && !force) {
    if (pendingRefreshCheck) {
      return pendingRefreshCheck
    }

    const promise = checkAndRefreshOAuthTokenIfNeededImpl(retryCount, force)
    pendingRefreshCheck = promise.finally(() => {
      pendingRefreshCheck = null
    })
    return pendingRefreshCheck
  }

  return checkAndRefreshOAuthTokenIfNeededImpl(retryCount, force)
}
```

**核心实现:**
```typescript
// utils/auth.ts:1447-1562
async function checkAndRefreshOAuthTokenIfNeededImpl(
  retryCount: number,
  force: boolean,
): Promise<boolean> {
  const MAX_RETRIES = 5

  // Step 1: 检查磁盘文件变更(多进程同步)
  await invalidateOAuthCacheIfDiskChanged()

  // Step 2: 获取当前token
  const tokens = getClaudeAIOAuthTokens()
  if (!force) {
    // 非强制模式：只有过期才刷新
    if (!tokens?.refreshToken || !isOAuthTokenExpired(tokens.expiresAt)) {
      return false
    }
  }

  // Step 3: 异步重新读取(检查其他进程是否已刷新)
  getClaudeAIOAuthTokens.cache?.clear?.()
  clearKeychainCache()
  const freshTokens = await getClaudeAIOAuthTokensAsync()
  if (!freshTokens?.refreshToken || !isOAuthTokenExpired(freshTokens.expiresAt)) {
    return false  // 其他进程已刷新
  }

  // Step 4: 获取文件锁(防止多进程并发刷新)
  const claudeDir = getClaudeConfigHomeDir()
  await mkdir(claudeDir, { recursive: true })

  let release
  try {
    release = await lockfile.lock(claudeDir)
  } catch (err) {
    if ((err as { code?: string }).code === 'ELOCKED') {
      // 锁被占用，等待重试
      if (retryCount < MAX_RETRIES) {
        await sleep(1000 + Math.random() * 1000)  // 随机延迟避免竞争
        return checkAndRefreshOAuthTokenIfNeededImpl(retryCount + 1, force)
      }
      return false
    }
    return false
  }

  try {
    // Step 5: 锁内再次检查(双重检查锁定模式)
    const lockedTokens = await getClaudeAIOAuthTokensAsync()
    if (!lockedTokens?.refreshToken || !isOAuthTokenExpired(lockedTokens.expiresAt)) {
      return false  // 其他进程在等待锁时已刷新
    }

    // Step 6: 执行刷新
    const refreshedTokens = await refreshOAuthToken(lockedTokens.refreshToken, {
      scopes: shouldUseClaudeAIAuth(lockedTokens.scopes)
        ? undefined  // Claude AI subscriber使用默认scope(允许扩展)
        : lockedTokens.scopes,  // Console用户保持原scope
    })
    saveOAuthTokensIfNeeded(refreshedTokens)

    // Step 7: 清理缓存
    getClaudeAIOAuthTokens.cache?.clear?.()
    clearKeychainCache()
    return true
  } catch (error) {
    // 刷新失败，检查是否有其他进程成功
    const currentTokens = await getClaudeAIOAuthTokensAsync()
    if (currentTokens && !isOAuthTokenExpired(currentTokens.expiresAt)) {
      return true  // 其他进程挽救了情况
    }
    return false
  } finally {
    await release()
  }
}
```

### 4.3 401错误处理

**服务端拒绝处理:**
```typescript
// utils/auth.ts:1360-1392
export function handleOAuth401Error(
  failedAccessToken: string,
): Promise<boolean> {
  // 同一token的401处理去重
  const pending = pending401Handlers.get(failedAccessToken)
  if (pending) return pending

  const promise = handleOAuth401ErrorImpl(failedAccessToken).finally(() => {
    pending401Handlers.delete(failedAccessToken)
  })
  pending401Handlers.set(failedAccessToken, promise)
  return promise
}

async function handleOAuth401ErrorImpl(
  failedAccessToken: string,
): Promise<boolean> {
  // 清理缓存重新读取
  clearOAuthTokenCache()
  const currentTokens = await getClaudeAIOAuthTokensAsync()

  if (!currentTokens?.refreshToken) {
    return false  // 无法恢复
  }

  // 检查keychain是否有不同token(其他tab已刷新)
  if (currentTokens.accessToken !== failedAccessToken) {
    logEvent('tengu_oauth_401_recovered_from_keychain', {})
    return true  // 已有有效token
  }

  // 强制刷新，绕过本地过期检查
  return checkAndRefreshOAuthTokenIfNeeded(0, true)
}
```

### 4.4 多进程同步

**文件变更检测:**
```typescript
// utils/auth.ts:1313-1336
let lastCredentialsMtimeMs = 0

async function invalidateOAuthCacheIfDiskChanged(): Promise<void> {
  try {
    const { mtimeMs } = await stat(
      join(getClaudeConfigHomeDir(), '.credentials.json')
    )
    if (mtimeMs !== lastCredentialsMtimeMs) {
      lastCredentialsMtimeMs = mtimeMs
      clearOAuthTokenCache()
    }
  } catch {
    // 文件不存在时清理memoize缓存
    getClaudeAIOAuthTokens.cache?.clear?.()
  }
}
```

**跨进程竞态解决:**
- 使用文件修改时间作为版本标记
- 其他进程写入后mtime变化触发缓存失效
- 结合文件锁实现完整的并发控制

---

## 5. 多认证源优先级

### 5.1 认证源类型

**AuthTokenSource枚举:**
```typescript
// utils/auth.ts:153-206
export function getAuthTokenSource() {
  // --bare模式: 仅允许settings flag的apiKeyHelper
  if (isBareMode()) {
    if (getConfiguredApiKeyHelper()) {
      return { source: 'apiKeyHelper' as const, hasToken: true }
    }
    return { source: 'none' as const, hasToken: false }
  }

  // 优先级顺序:
  // 1. ANTHROPIC_AUTH_TOKEN (非managed context)
  if (process.env.ANTHROPIC_AUTH_TOKEN && !isManagedOAuthContext()) {
    return { source: 'ANTHROPIC_AUTH_TOKEN' as const, hasToken: true }
  }

  // 2. CLAUDE_CODE_OAUTH_TOKEN
  if (process.env.CLAUDE_CODE_OAUTH_TOKEN) {
    return { source: 'CLAUDE_CODE_OAUTH_TOKEN' as const, hasToken: true }
  }

  // 3. OAuth Token from File Descriptor
  const oauthTokenFromFd = getOAuthTokenFromFileDescriptor()
  if (oauthTokenFromFd) {
    if (process.env.CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR) {
      return {
        source: 'CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR' as const,
        hasToken: true,
      }
    }
    return { source: 'CCR_OAUTH_TOKEN_FILE' as const, hasToken: true }
  }

  // 4. apiKeyHelper (非managed context)
  const apiKeyHelper = getConfiguredApiKeyHelper()
  if (apiKeyHelper && !isManagedOAuthContext()) {
    return { source: 'apiKeyHelper' as const, hasToken: true }
  }

  // 5. Claude.ai OAuth (通过/login)
  const oauthTokens = getClaudeAIOAuthTokens()
  if (shouldUseClaudeAIAuth(oauthTokens?.scopes) && oauthTokens?.accessToken) {
    return { source: 'claude.ai' as const, hasToken: true }
  }

  return { source: 'none' as const, hasToken: false }
}
```

### 5.2 API Key获取优先级

**ApiKeySource枚举:**
```typescript
// utils/auth.ts:208-213
export type ApiKeySource =
  | 'ANTHROPIC_API_KEY'
  | 'apiKeyHelper'
  | '/login managed key'
  | 'none'
```

**获取逻辑:**
```typescript
// utils/auth.ts:226-347
export function getAnthropicApiKeyWithSource(
  opts: { skipRetrievingKeyFromApiKeyHelper?: boolean } = {},
): { key: null | string; source: ApiKeySource } {
  // --bare模式: 仅环境变量或settings flag的apiKeyHelper
  if (isBareMode()) {
    if (process.env.ANTHROPIC_API_KEY) {
      return { key: process.env.ANTHROPIC_API_KEY, source: 'ANTHROPIC_API_KEY' }
    }
    if (getConfiguredApiKeyHelper()) {
      return {
        key: opts.skipRetrievingKeyFromApiKeyHelper
          ? null
          : getApiKeyFromApiKeyHelperCached(),
        source: 'apiKeyHelper',
      }
    }
    return { key: null, source: 'none' }
  }

  // Homespace环境: 不使用ANTHROPIC_API_KEY
  const apiKeyEnv = isRunningOnHomespace()
    ? undefined
    : process.env.ANTHROPIC_API_KEY

  // --print模式优先使用环境变量
  if (preferThirdPartyAuthentication() && apiKeyEnv) {
    return { key: apiKeyEnv, source: 'ANTHROPIC_API_KEY' }
  }

  // CI环境
  if (isEnvTruthy(process.env.CI) || process.env.NODE_ENV === 'test') {
    // 文件描述符优先
    const apiKeyFromFd = getApiKeyFromFileDescriptor()
    if (apiKeyFromFd) {
      return { key: apiKeyFromFd, source: 'ANTHROPIC_API_KEY' }
    }

    // 必须有环境变量或OAuth token
    if (!apiKeyEnv && !process.env.CLAUDE_CODE_OAUTH_TOKEN && ...) {
      throw new Error('ANTHROPIC_API_KEY or CLAUDE_CODE_OAUTH_TOKEN env var is required')
    }

    if (apiKeyEnv) {
      return { key: apiKeyEnv, source: 'ANTHROPIC_API_KEY' }
    }

    return { key: null, source: 'none' }
  }

  // 常规环境: 检查环境变量是否已批准
  if (apiKeyEnv && isCustomApiKeyApproved(apiKeyEnv)) {
    return { key: apiKeyEnv, source: 'ANTHROPIC_API_KEY' }
  }

  // 文件描述符
  const apiKeyFromFd = getApiKeyFromFileDescriptor()
  if (apiKeyFromFd) {
    return { key: apiKeyFromFd, source: 'ANTHROPIC_API_KEY' }
  }

  // apiKeyHelper
  const apiKeyHelperCommand = getConfiguredApiKeyHelper()
  if (apiKeyHelperCommand) {
    return { key: getApiKeyFromApiKeyHelperCached(), source: 'apiKeyHelper' }
  }

  // /login管理的key (keychain或config)
  const apiKeyFromConfigOrMacOSKeychain = getApiKeyFromConfigOrMacOSKeychain()
  if (apiKeyFromConfigOrMacOSKeychain) {
    return apiKeyFromConfigOrMacOSKeychain
  }

  return { key: null, source: 'none' }
}
```

### 5.3 Managed OAuth Context

**概念:** CCR和Claude Desktop spawn的场景，不应使用用户个人配置

```typescript
// utils/auth.ts:91-96
function isManagedOAuthContext(): boolean {
  return (
    isEnvTruthy(process.env.CLAUDE_CODE_REMOTE) ||
    process.env.CLAUDE_CODE_ENTRYPOINT === 'claude-desktop'
  )
}
```

**作用:**
- 防止个人`~/.claude/settings.json`干扰managed session
- 避免错误组织匹配
- 确保使用正确的OAuth token

### 5.4 Third-Party Services

**Bedrock/Vertex/Foundry判断:**
```typescript
// utils/auth.ts:1732-1738
export function isUsing3PServices(): boolean {
  return !!(
    isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK) ||
    isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX) ||
    isEnvTruthy(process.env.CLAUDE_CODE_USE_FOUNDRY)
  )
}
```

**对认证的影响:**
```typescript
// utils/auth.ts:115-148
export function isAnthropicAuthEnabled(): boolean {
  // --bare: 仅API-key，无OAuth
  if (isBareMode()) return false

  // SSH remote特殊情况
  if (process.env.ANTHROPIC_UNIX_SOCKET) {
    return !!process.env.CLAUDE_CODE_OAUTH_TOKEN
  }

  const is3P = isUsing3PServices()

  // 外部API key或auth token存在时禁用Anthropic auth
  const hasExternalAuthToken =
    process.env.ANTHROPIC_AUTH_TOKEN ||
    apiKeyHelper ||
    process.env.CLAUDE_CODE_API_KEY_FILE_DESCRIPTOR

  const hasExternalApiKey =
    apiKeySource === 'ANTHROPIC_API_KEY' || apiKeySource === 'apiKeyHelper'

  const shouldDisableAuth =
    is3P ||
    (hasExternalAuthToken && !isManagedOAuthContext()) ||
    (hasExternalApiKey && !isManagedOAuthContext())

  return !shouldDisableAuth
}
```

---

## 6. 用户使用指南

### 6.1 登录命令

**命令:** `claude auth login`

**选项:**
```bash
# 使用claude.ai登录 (默认)
claude auth login

# 使用Console登录
claude auth login --console

# 使用claude.ai登录 (显式)
claude auth login --claudeai

# 预填充邮箱
claude auth login --email user@example.com

# SSO登录
claude auth login --sso
```

**cli/handlers/auth.ts实现:**
```typescript
// cli/handlers/auth.ts:112-230
export async function authLogin({
  email,
  sso,
  console: useConsole,
  claudeai,
}): Promise<void> {
  // 选项互斥检查
  if (useConsole && claudeai) {
    process.stderr.write('Error: --console and --claudeai cannot be used together.\n')
    process.exit(1)
  }

  // 确定登录方式
  const loginWithClaudeAi = settings.forceLoginMethod
    ? settings.forceLoginMethod === 'claudeai'
    : !useConsole

  // 快速路径: 环境变量refresh token
  const envRefreshToken = process.env.CLAUDE_CODE_OAUTH_REFRESH_TOKEN
  if (envRefreshToken) {
    const envScopes = process.env.CLAUDE_CODE_OAUTH_SCOPES
    if (!envScopes) {
      process.stderr.write('CLAUDE_CODE_OAUTH_SCOPES is required...\n')
      process.exit(1)
    }

    const scopes = envScopes.split(/\s+/).filter(Boolean)
    const tokens = await refreshOAuthToken(envRefreshToken, { scopes })
    await installOAuthTokens(tokens)

    // 验证组织
    const orgResult = await validateForceLoginOrg()
    if (!orgResult.valid) {
      process.stderr.write(orgResult.message + '\n')
      process.exit(1)
    }

    process.stdout.write('Login successful.\n')
    process.exit(0)
  }

  // 标准OAuth流程
  const oauthService = new OAuthService()
  try {
    const result = await oauthService.startOAuthFlow(
      async url => {
        process.stdout.write('Opening browser to sign in...\n')
        process.stdout.write(`If the browser didn't open, visit: ${url}\n`)
      },
      {
        loginWithClaudeAi,
        loginHint: email,
        loginMethod: sso ? 'sso' : undefined,
        orgUUID: settings.forceLoginOrgUUID,
      },
    )

    await installOAuthTokens(result)
    ...
  } finally {
    oauthService.cleanup()
  }
}
```

### 6.2 状态查看

**命令:** `claude auth status`

**输出格式:**
```typescript
// cli/handlers/auth.ts:232-318
export async function authStatus(opts: { json?: boolean; text?: boolean }): Promise<void> {
  const { source: authTokenSource, hasToken } = getAuthTokenSource()
  const { source: apiKeySource } = getAnthropicApiKeyWithSource()
  const oauthAccount = getOauthAccountInfo()
  const subscriptionType = getSubscriptionType()

  // 认证方法判断
  let authMethod: string = 'none'
  if (using3P) {
    authMethod = 'third_party'
  } else if (authTokenSource === 'claude.ai') {
    authMethod = 'claude.ai'
  } else if (authTokenSource === 'apiKeyHelper') {
    authMethod = 'api_key_helper'
  } else if (authTokenSource !== 'none') {
    authMethod = 'oauth_token'
  } else if (apiKeySource === 'ANTHROPIC_API_KEY') {
    authMethod = 'api_key'
  } else if (apiKeySource === '/login managed key') {
    authMethod = 'claude.ai'
  }

  // JSON输出
  if (!opts.text) {
    const output: Record<string, string | boolean | null> = {
      loggedIn,
      authMethod,
      apiProvider,
    }
    if (authMethod === 'claude.ai') {
      output.email = oauthAccount?.emailAddress ?? null
      output.orgId = oauthAccount?.organizationUuid ?? null
      output.subscriptionType = subscriptionType ?? null
    }
    process.stdout.write(jsonStringify(output, null, 2) + '\n')
  }

  process.exit(loggedIn ? 0 : 1)
}
```

### 6.3 登出

**命令:** `claude auth logout`

**实现:**
```typescript
// cli/handlers/auth.ts:321-330
export async function authLogout(): Promise<void> {
  try {
    await performLogout({ clearOnboarding: false })
  } catch {
    process.stderr.write('Failed to log out.\n')
    process.exit(1)
  }
  process.stdout.write('Successfully logged out from your Anthropic account.\n')
  process.exit(0)
}
```

### 6.4 环境变量配置

**关键环境变量:**

| 变量名 | 用途 | 示例 |
|--------|------|------|
| `ANTHROPIC_API_KEY` | API密钥 | `sk-ant-...` |
| `CLAUDE_CODE_OAUTH_TOKEN` | OAuth访问令牌 | 用于远程/CI场景 |
| `CLAUDE_CODE_OAUTH_REFRESH_TOKEN` | 刷新令牌 | 用于快速登录 |
| `CLAUDE_CODE_OAUTH_SCOPES` | OAuth scopes | `user:inference user:profile` |
| `CLAUDE_CODE_USE_BEDROCK` | 使用AWS Bedrock | `true` |
| `CLAUDE_CODE_USE_VERTEX` | 使用Google Vertex | `true` |
| `CLAUDE_CODE_USE_FOUNDRY` | 使用Foundry | `true` |

**使用示例:**
```bash
# CI环境
export ANTHROPIC_API_KEY="sk-ant-..."
export CLAUDE_CODE_OAUTH_TOKEN="..."

# 快速登录(有refresh token)
export CLAUDE_CODE_OAUTH_REFRESH_TOKEN="..."
export CLAUDE_CODE_OAUTH_SCOPES="user:inference user:profile"

# 第三方服务
export CLAUDE_CODE_USE_BEDROCK=true
export AWS_PROFILE=...
```

---

## 7. 从零实现

### 7.1 最小OAuth客户端实现

**Step 1: PKCE生成**
```typescript
// crypto.ts
import { createHash, randomBytes } from 'crypto'

function base64URLEncode(buffer: Buffer): string {
  return buffer.toString('base64')
    .replace(/\+/g, '-')
    .replace(/\//g, '_')
    .replace(/=/g, '')
}

export function generateCodeVerifier(): string {
  return base64URLEncode(randomBytes(32))
}

export function generateCodeChallenge(verifier: string): string {
  return base64URLEncode(
    createHash('sha256').update(verifier).digest()
  )
}

export function generateState(): string {
  return base64URLEncode(randomBytes(32))
}
```

**Step 2: 回调监听器**
```typescript
// listener.ts
import { createServer } from 'http'

export class AuthCodeListener {
  private server = createServer()
  private port = 0
  private resolver: ((code: string) => void) | null = null
  private expectedState: string | null = null

  async start(): Promise<number> {
    return new Promise(resolve => {
      this.server.listen(0, 'localhost', () => {
        this.port = (this.server.address() as any).port
        resolve(this.port)
      })
    })
  }

  async waitForCode(state: string): Promise<string> {
    this.expectedState = state
    return new Promise(resolve => {
      this.resolver = resolve
      this.server.on('request', (req, res) => {
        const url = new URL(req.url!, `http://localhost:${this.port}`)
        const code = url.searchParams.get('code')
        const state = url.searchParams.get('state')

        if (state === this.expectedState && code) {
          res.writeHead(302, { Location: 'https://claude.ai/oauth/success' })
          res.end()
          this.resolver!(code)
        }
      })
    })
  }

  close() {
    this.server.close()
  }
}
```

**Step 3: OAuth服务**
```typescript
// oauth.ts
import axios from 'axios'

const CONFIG = {
  authorizeUrl: 'https://claude.com/cai/oauth/authorize',
  tokenUrl: 'https://platform.claude.com/v1/oauth/token',
  clientId: '9d1c250a-e61b-44d9-88ed-5944d1962f5e',
  scopes: ['user:profile', 'user:inference', 'user:sessions:claude_code'],
}

export class OAuthService {
  private verifier = generateCodeVerifier()
  private listener = new AuthCodeListener()

  async login(): Promise<{ accessToken: string; refreshToken: string }> {
    const port = await this.listener.start()
    const challenge = generateCodeChallenge(this.verifier)
    const state = generateState()

    // 构建授权URL
    const authUrl = new URL(CONFIG.authorizeUrl)
    authUrl.searchParams.set('client_id', CONFIG.clientId)
    authUrl.searchParams.set('response_type', 'code')
    authUrl.searchParams.set('redirect_uri', `http://localhost:${port}/callback`)
    authUrl.searchParams.set('code_challenge', challenge)
    authUrl.searchParams.set('code_challenge_method', 'S256')
    authUrl.searchParams.set('state', state)
    authUrl.searchParams.set('scope', CONFIG.scopes.join(' '))

    // 打开浏览器
    console.log(`请访问: ${authUrl.toString()}`)

    // 等待回调
    const code = await this.listener.waitForCode(state)

    // 交换token
    const tokens = await this.exchangeCode(code, port)

    this.listener.close()
    return tokens
  }

  private async exchangeCode(code: string, port: number) {
    const response = await axios.post(CONFIG.tokenUrl, {
      grant_type: 'authorization_code',
      code,
      redirect_uri: `http://localhost:${port}/callback`,
      client_id: CONFIG.clientId,
      code_verifier: this.verifier,
    })

    return {
      accessToken: response.data.access_token,
      refreshToken: response.data.refresh_token,
      expiresAt: Date.now() + response.data.expires_in * 1000,
    }
  }
}
```

### 7.2 Token刷新实现

```typescript
// refresh.ts
export async function refreshToken(refreshToken: string) {
  const response = await axios.post(CONFIG.tokenUrl, {
    grant_type: 'refresh_token',
    refresh_token: refreshToken,
    client_id: CONFIG.clientId,
    scope: CONFIG.scopes.join(' '),
  })

  return {
    accessToken: response.data.access_token,
    refreshToken: response.data.refresh_token ?? refreshToken,
    expiresAt: Date.now() + response.data.expires_in * 1000,
  }
}

// 过期检测
export function isExpired(expiresAt: number): boolean {
  const buffer = 5 * 60 * 1000  // 5分钟缓冲
  return Date.now() + buffer >= expiresAt
}

// 自动刷新检查
export async function checkAndRefresh(tokens: any): Promise<any> {
  if (!isExpired(tokens.expiresAt)) {
    return tokens
  }

  const newTokens = await refreshToken(tokens.refreshToken)
  // 存储 newTokens
  return newTokens
}
```

### 7.3 安全存储实现

```typescript
// storage.ts
import { writeFile, readFile } from 'fs/promises'
import { join } from 'path'
import { homedir } from 'os'

const STORAGE_PATH = join(homedir(), '.claude', '.credentials.json')

export async function saveTokens(tokens: any) {
  await writeFile(STORAGE_PATH, JSON.stringify({
    claudeAiOauth: {
      accessToken: tokens.accessToken,
      refreshToken: tokens.refreshToken,
      expiresAt: tokens.expiresAt,
      scopes: tokens.scopes,
    }
  }), { mode: 0o600 })  // 仅用户可读写
}

export async function loadTokens(): Promise<any | null> {
  try {
    const data = await readFile(STORAGE_PATH, 'utf-8')
    const parsed = JSON.parse(data)
    return parsed.claudeAiOauth
  } catch {
    return null
  }
}
```

### 7.4 完整流程示例

```typescript
// main.ts
async function main() {
  // 1. 检查现有token
  const existing = await loadTokens()

  if (existing) {
    // 2. 检查是否过期
    if (isExpired(existing.expiresAt)) {
      // 3. 刷新token
      const refreshed = await refreshToken(existing.refreshToken)
      await saveTokens(refreshed)
      return refreshed.accessToken
    }
    return existing.accessToken
  }

  // 4. 新登录
  const oauth = new OAuthService()
  const tokens = await oauth.login()
  await saveTokens(tokens)
  return tokens.accessToken
}

// 使用token进行API调用
async function callAPI(accessToken: string) {
  const response = await axios.post(
    'https://api.anthropic.com/v1/messages',
    { model: 'claude-3-5-sonnet-20241022', max_tokens: 1024, messages: [...] },
    { headers: { Authorization: `Bearer ${accessToken}` } }
  )
  return response.data
}
```

---

## 附录

### A. OAuth Scope说明

| Scope | 说明 |
|-------|------|
| `user:profile` | 用户基本信息 |
| `user:inference` | API推理权限 |
| `user:sessions:claude_code` | Claude Code会话 |
| `user:mcp_servers` | MCP服务器访问 |
| `user:file_upload` | 文件上传权限 |
| `org:create_api_key` | 创建API密钥(Console) |

### B. 订阅类型

| 类型 | 说明 |
|------|------|
| `max` | Claude Max订阅 |
| `pro` | Claude Pro订阅 |
| `team` | Claude Team订阅 |
| `enterprise` | Claude Enterprise订阅 |

### C. 关键文件索引

| 文件路径 | 功能 |
|----------|------|
| `services/oauth/index.ts` | OAuthService主类 |
| `services/oauth/client.ts` | Token交换/刷新/Profile获取 |
| `services/oauth/crypto.ts` | PKCE加密工具 |
| `services/oauth/auth-code-listener.ts` | 本地回调监听器 |
| `services/oauth/getOauthProfile.ts` | Profile API封装 |
| `constants/oauth.ts` | OAuth配置常量 |
| `cli/handlers/auth.ts` | 认证命令处理 |
| `utils/auth.ts` | 认证工具/Token刷新 |
| `utils/secureStorage/*.ts` | 安全存储实现 |

---

*分析完成: 2026-04-02*