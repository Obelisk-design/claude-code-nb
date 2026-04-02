# 多后端API支持详解

**分析日期:** 2026-04-02

## 1. 多后端架构概述

### 1.1 架构设计理念

Claude Code CLI 采用**后端抽象层**设计,支持四种不同的 API 后端:

```
┌─────────────────────────────────────────────────────────────┐
│                    Claude Code CLI                           │
│                   (统一调用接口)                              │
├─────────────────────────────────────────────────────────────┤
│              services/api/client.ts                          │
│           (getAnthropicClient 工厂函数)                      │
├────────────┬────────────┬────────────┬────────────┬─────────┤
│ Anthropic  │   Bedrock  │   Vertex   │  Foundry   │  其他   │
│  API       │   (AWS)    │  (GCP)     │  (Azure)   │ (代理)  │
└────────────┴────────────┴────────────┴────────────┴─────────┘
```

**核心设计原则:**

1. **统一接口**: 所有后端返回 `Anthropic` 类型,保持 API 调用一致性
2. **动态加载**: SDK 模块按需导入,减少启动时内存占用
3. **配置驱动**: 通过环境变量切换后端,无需代码修改
4. **认证抽象**: 每个后端独立的认证刷新机制

### 1.2 后端类型定义

```typescript
// utils/model/providers.ts:4
export type APIProvider = 'firstParty' | 'bedrock' | 'vertex' | 'foundry'

export function getAPIProvider(): APIProvider {
  return isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK)
    ? 'bedrock'
    : isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX)
      ? 'vertex'
      : isEnvTruthy(process.env.CLAUDE_CODE_USE_FOUNDRY)
        ? 'foundry'
        : 'firstParty'
}
```

**优先级顺序:** Bedrock > Vertex > Foundry > FirstParty

### 1.3 环境变量矩阵

| 后端 | 启用变量 | 认证变量 | 区域变量 |
|------|----------|----------|----------|
| Anthropic API | - | `ANTHROPIC_API_KEY` | - |
| AWS Bedrock | `CLAUDE_CODE_USE_BEDROCK` | AWS 凭证链 | `AWS_REGION` |
| Google Vertex | `CLAUDE_CODE_USE_VERTEX` | GCP ADC | `CLOUD_ML_REGION` |
| Azure Foundry | `CLAUDE_CODE_USE_FOUNDRY` | Azure AD | `ANTHROPIC_FOUNDRY_RESOURCE` |

---

## 2. Anthropic API (FirstParty)

### 2.1 客户端创建流程

```typescript
// services/api/client.ts:300-316
const clientConfig: ConstructorParameters<typeof Anthropic>[0] = {
  apiKey: isClaudeAISubscriber() ? null : apiKey || getAnthropicApiKey(),
  authToken: isClaudeAISubscriber()
    ? getClaudeAIOAuthTokens()?.accessToken
    : undefined,
  // OAuth staging 环境特殊处理
  ...(process.env.USER_TYPE === 'ant' &&
  isEnvTruthy(process.env.USE_STAGING_OAUTH)
    ? { baseURL: getOauthConfig().BASE_API_URL }
    : {}),
  ...ARGS,
  ...(isDebugToStdErr() && { logger: createStderrLogger() }),
}

return new Anthropic(clientConfig)
```

**认证方式选择逻辑:**

```typescript
// utils/auth.ts:98-149
export function isAnthropicAuthEnabled(): boolean {
  // 1. --bare 模式: 仅 API key,禁用 OAuth
  if (isBareMode()) return false

  // 2. SSH 远程模式: 检查 Unix socket 代理
  if (process.env.ANTHROPIC_UNIX_SOCKET) {
    return !!process.env.CLAUDE_CODE_OAUTH_TOKEN
  }

  // 3. 第三方后端: 禁用 Anthropic OAuth
  const is3P =
    isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK) ||
    isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX) ||
    isEnvTruthy(process.env.CLAUDE_CODE_USE_FOUNDRY)

  // 4. 外部 API key: 优先使用
  const hasExternalApiKey =
    apiKeySource === 'ANTHROPIC_API_KEY' || apiKeySource === 'apiKeyHelper'

  return !(is3P || hasExternalApiKey)
}
```

### 2.2 API Key 来源优先级

```typescript
// utils/auth.ts:226-300
export function getAnthropicApiKeyWithSource(opts): { key, source } {
  // 1. Bare 模式: 仅环境变量或 apiKeyHelper
  if (isBareMode()) {
    if (process.env.ANTHROPIC_API_KEY) {
      return { key: process.env.ANTHROPIC_API_KEY, source: 'ANTHROPIC_API_KEY' }
    }
    if (getConfiguredApiKeyHelper()) {
      return { key: getApiKeyFromApiKeyHelperCached(), source: 'apiKeyHelper' }
    }
    return { key: null, source: 'none' }
  }

  // 2. Homespace 环境: 禁用环境变量 API key
  const apiKeyEnv = isRunningOnHomespace()
    ? undefined
    : process.env.ANTHROPIC_API_KEY

  // 3. --print 模式: 优先环境变量
  if (preferThirdPartyAuthentication() && apiKeyEnv) {
    return { key: apiKeyEnv, source: 'ANTHROPIC_API_KEY' }
  }

  // 4. CI/测试环境: 强制要求环境变量
  if (isEnvTruthy(process.env.CI) || process.env.NODE_ENV === 'test') {
    // 检查文件描述符传递的 key
    const apiKeyFromFd = getApiKeyFromFileDescriptor()
    if (apiKeyFromFd) {
      return { key: apiKeyFromFd, source: 'ANTHROPIC_API_KEY' }
    }
    if (!apiKeyEnv && !process.env.CLAUDE_CODE_OAUTH_TOKEN) {
      throw new Error('ANTHROPIC_API_KEY or CLAUDE_CODE_OAUTH_TOKEN required')
    }
  }

  // 5. 正常模式: 环境变量 > apiKeyHelper > macOS keychain > /login
  // (详见完整流程...)
}
```

### 2.3 自定义 Headers 支持

```typescript
// services/api/client.ts:330-354
function getCustomHeaders(): Record<string, string> {
  const customHeaders: Record<string, string> = {}
  const customHeadersEnv = process.env.ANTHROPIC_CUSTOM_HEADERS

  if (!customHeadersEnv) return customHeaders

  // 支持多行格式,每行一个 Header
  const headerStrings = customHeadersEnv.split(/\n|\r\n/)

  for (const headerString of headerStrings) {
    if (!headerString.trim()) continue

    // 解析 "Name: Value" 格式 (curl 风格)
    const colonIdx = headerString.indexOf(':')
    if (colonIdx === -1) continue
    const name = headerString.slice(0, colonIdx).trim()
    const value = headerString.slice(colonIdx + 1).trim()
    if (name) {
      customHeaders[name] = value
    }
  }

  return customHeaders
}
```

**使用示例:**
```bash
# 添加自定义 Authorization header
ANTHROPIC_CUSTOM_HEADERS="Authorization: Bearer custom-token
X-Custom-Header: value"
```

---

## 3. AWS Bedrock

### 3.1 Bedrock 客户端创建

```typescript
// services/api/client.ts:153-189
if (isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK)) {
  const { AnthropicBedrock } = await import('@anthropic-ai/bedrock-sdk')

  // 小模型区域覆盖 (Haiku 可使用不同区域)
  const awsRegion =
    model === getSmallFastModel() &&
    process.env.ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION
      ? process.env.ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION
      : getAWSRegion()

  const bedrockArgs = {
    ...ARGS,
    awsRegion,

    // 测试/代理场景: 跳过认证
    ...(isEnvTruthy(process.env.CLAUDE_CODE_SKIP_BEDROCK_AUTH) && {
      skipAuth: true,
    }),

    // API Key 认证 (Bearer Token)
    ...(process.env.AWS_BEARER_TOKEN_BEDROCK && {
      skipAuth: true,
      defaultHeaders: {
        Authorization: `Bearer ${process.env.AWS_BEARER_TOKEN_BEDROCK}`,
      },
    }),
  }

  // 标准 AWS 凭证认证
  if (!isEnvTruthy(process.env.CLAUDE_CODE_SKIP_BEDROCK_AUTH)) {
    const cachedCredentials = await refreshAndGetAwsCredentials()
    if (cachedCredentials) {
      bedrockArgs.awsAccessKey = cachedCredentials.accessKeyId
      bedrockArgs.awsSecretKey = cachedCredentials.secretAccessKey
      bedrockArgs.awsSessionToken = cachedCredentials.sessionToken
    }
  }

  return new AnthropicBedrock(bedrockArgs) as unknown as Anthropic
}
```

### 3.2 AWS 凭证管理

```typescript
// utils/aws.ts:5-22
export type AwsCredentials = {
  AccessKeyId: string
  SecretAccessKey: string
  SessionToken: string
  Expiration?: string
}

export type AwsStsOutput = {
  Credentials: AwsCredentials
}

// 凭证验证
export function isValidAwsStsOutput(obj: unknown): obj is AwsStsOutput {
  if (!obj || typeof obj !== 'object') return false

  const credentials = obj.Credentials as Record<string, unknown>
  return (
    typeof credentials.AccessKeyId === 'string' &&
    typeof credentials.SecretAccessKey === 'string' &&
    typeof credentials.SessionToken === 'string' &&
    credentials.AccessKeyId.length > 0 &&
    credentials.SecretAccessKey.length > 0 &&
    credentials.SessionToken.length > 0
  )
}

// 检查 STS Caller Identity (验证 AWS 配置)
export async function checkStsCallerIdentity(): Promise<void> {
  const { STSClient, GetCallerIdentityCommand } = await import(
    '@aws-sdk/client-sts'
  )
  await new STSClient().send(new GetCallerIdentityCommand({}))
}

// 清除 INI 缓存 (刷新 ~/.aws/credentials)
export async function clearAwsIniCache(): Promise<void> {
  const { fromIni } = await import('@aws-sdk/credential-providers')
  const iniProvider = fromIni({ ignoreCache: true })
  await iniProvider()
}
```

### 3.3 AWS 凭证刷新机制

```typescript
// utils/auth.ts:787-807
export const refreshAndGetAwsCredentials = memoizeWithTTLAsync(
  async (): Promise<{
    accessKeyId: string
    secretAccessKey: string
    sessionToken: string
  } | null> => {
    // 1. 运行 awsAuthRefresh 命令 (如配置)
    const refreshed = await runAwsAuthRefresh()

    // 2. 从 awsCredentialExport 获取凭证
    const credentials = await getAwsCredsFromCredentialExport()

    // 3. 清除 INI 缓存确保新鲜凭证
    if (refreshed || credentials) {
      await clearAwsIniCache()
    }

    return credentials
  },
  DEFAULT_AWS_STS_TTL, // TTL 缓存
)
```

**AWS 凭证导出流程:**

```typescript
// utils/auth.ts:701-780
async function getAwsCredsFromCredentialExport() {
  const awsCredentialExport = getConfiguredAwsCredentialExport()
  if (!awsCredentialExport) return null

  // 安全检查: 项目配置需要 workspace trust
  if (isAwsCredentialExportFromProjectSettings()) {
    const hasTrust = checkHasTrustDialogAccepted()
    if (!hasTrust) {
      logEvent('tengu_awsCredentialExport_missing_trust', {})
      return null
    }
  }

  try {
    // 先检查 STS caller identity
    await checkStsCallerIdentity()
    return null // 已有有效凭证,无需导出
  } catch {
    // 执行导出命令
    const result = await execa(awsCredentialExport, { shell: true })
    const awsOutput = jsonParse(result.stdout.trim())

    if (!isValidAwsStsOutput(awsOutput)) {
      throw new Error('Invalid AWS STS output structure')
    }

    return {
      accessKeyId: awsOutput.Credentials.AccessKeyId,
      secretAccessKey: awsOutput.Credentials.SecretAccessKey,
      sessionToken: awsOutput.Credentials.SessionToken,
    }
  }
}
```

### 3.4 Bedrock Inference Profiles

```typescript
// utils/model/bedrock.ts:7-41
export const getBedrockInferenceProfiles = memoize(async function (): Promise<
  string[]
> {
  const [client, { ListInferenceProfilesCommand }] = await Promise.all([
    createBedrockClient(),
    import('@aws-sdk/client-bedrock'),
  ])

  const allProfiles = []
  let nextToken: string | undefined

  do {
    const command = new ListInferenceProfilesCommand({
      ...(nextToken && { nextToken }),
      typeEquals: 'SYSTEM_DEFINED',
    })
    const response = await client.send(command)

    if (response.inferenceProfileSummaries) {
      allProfiles.push(...response.inferenceProfileSummaries)
    }
    nextToken = response.nextToken
  } while (nextToken)

  // 过滤 Anthropic 模型
  return allProfiles
    .filter(profile => profile.inferenceProfileId?.includes('anthropic'))
    .map(profile => profile.inferenceProfileId)
    .filter(Boolean) as string[]
})
```

**Bedrock 客户端创建:**

```typescript
// utils/model/bedrock.ts:50-94
async function createBedrockClient() {
  const { BedrockClient } = await import('@aws-sdk/client-bedrock')
  const region = getAWSRegion()
  const skipAuth = isEnvTruthy(process.env.CLAUDE_CODE_SKIP_BEDROCK_AUTH)

  const clientConfig = {
    region,
    ...(process.env.ANTHROPIC_BEDROCK_BASE_URL && {
      endpoint: process.env.ANTHROPIC_BEDROCK_BASE_URL,
    }),
    ...(await getAWSClientProxyConfig()),
    ...(skipAuth && {
      // 无认证配置
      requestHandler: new NodeHttpHandler(),
      httpAuthSchemes: [{
        schemeId: 'smithy.api#noAuth',
        identityProvider: () => async () => ({}),
        signer: new NoAuthSigner(),
      }],
    }),
  }

  if (!skipAuth && !process.env.AWS_BEARER_TOKEN_BEDROCK) {
    const cachedCredentials = await refreshAndGetAwsCredentials()
    if (cachedCredentials) {
      clientConfig.credentials = {
        accessKeyId: cachedCredentials.accessKeyId,
        secretAccessKey: cachedCredentials.secretAccessKey,
        sessionToken: cachedCredentials.sessionToken,
      }
    }
  }

  return new BedrockClient(clientConfig)
}
```

### 3.5 Bedrock 区域前缀处理

```typescript
// utils/model/bedrock.ts:189-235
const BEDROCK_REGION_PREFIXES = ['us', 'eu', 'apac', 'global'] as const

export type BedrockRegionPrefix = (typeof BEDROCK_REGION_PREFIXES)[number]

// 提取区域前缀
export function getBedrockRegionPrefix(modelId: string): BedrockRegionPrefix | undefined {
  const effectiveModelId = extractModelIdFromArn(modelId)

  for (const prefix of BEDROCK_REGION_PREFIXES) {
    if (effectiveModelId.startsWith(`${prefix}.anthropic.`)) {
      return prefix
    }
  }
  return undefined
}

// 应用区域前缀
export function applyBedrockRegionPrefix(
  modelId: string,
  prefix: BedrockRegionPrefix,
): string {
  const existingPrefix = getBedrockRegionPrefix(modelId)
  if (existingPrefix) {
    return modelId.replace(`${existingPrefix}.`, `${prefix}.`)
  }

  if (isFoundationModel(modelId)) {
    return `${prefix}.${modelId}`
  }

  return modelId
}

// 判断是否为 Foundation Model
export function isFoundationModel(modelId: string): boolean {
  return modelId.startsWith('anthropic.')
}
```

**示例:**
```
输入: "anthropic.claude-sonnet-4-5-v1:0"
输出: "eu.anthropic.claude-sonnet-4-5-v1:0" (添加 eu 前缀)

输入: "us.anthropic.claude-opus-4-6-v1"
输出: "eu.anthropic.claude-opus-4-6-v1" (替换 us 为 eu)
```

---

## 4. Google Vertex AI

### 4.1 Vertex 客户端创建

```typescript
// services/api/client.ts:221-297
if (isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX)) {
  // 刷新 GCP 凭证
  if (!isEnvTruthy(process.env.CLAUDE_CODE_SKIP_VERTEX_AUTH)) {
    await refreshGcpCredentialsIfNeeded()
  }

  const [{ AnthropicVertex }, { GoogleAuth }] = await Promise.all([
    import('@anthropic-ai/vertex-sdk'),
    import('google-auth-library'),
  ])

  // 检查项目环境变量
  const hasProjectEnvVar =
    process.env['GCLOUD_PROJECT'] ||
    process.env['GOOGLE_CLOUD_PROJECT'] ||
    process.env['gcloud_project'] ||
    process.env['google_cloud_project']

  const hasKeyFile =
    process.env['GOOGLE_APPLICATION_CREDENTIALS'] ||
    process.env['google_application_credentials']

  const googleAuth = isEnvTruthy(process.env.CLAUDE_CODE_SKIP_VERTEX_AUTH)
    ? ({
        // Mock GoogleAuth for testing
        getClient: () => ({ getRequestHeaders: () => ({}) }),
      } as unknown as GoogleAuth)
    : new GoogleAuth({
        scopes: ['https://www.googleapis.com/auth/cloud-platform'],
        // 仅在没有其他配置时使用 ANTHROPIC_VERTEX_PROJECT_ID
        ...(hasProjectEnvVar || hasKeyFile
          ? {}
          : { projectId: process.env.ANTHROPIC_VERTEX_PROJECT_ID }),
      })

  const vertexArgs = {
    ...ARGS,
    region: getVertexRegionForModel(model),
    googleAuth,
    ...(isDebugToStdErr() && { logger: createStderrLogger() }),
  }

  return new AnthropicVertex(vertexArgs) as unknown as Anthropic
}
```

### 4.2 Vertex 区域配置

```typescript
// utils/envUtils.ts:149-183
// 模型特定的区域覆盖变量
const VERTEX_REGION_OVERRIDES: ReadonlyArray<[string, string]> = [
  ['claude-haiku-4-5', 'VERTEX_REGION_CLAUDE_HAIKU_4_5'],
  ['claude-3-5-haiku', 'VERTEX_REGION_CLAUDE_3_5_HAIKU'],
  ['claude-3-5-sonnet', 'VERTEX_REGION_CLAUDE_3_5_SONNET'],
  ['claude-3-7-sonnet', 'VERTEX_REGION_CLAUDE_3_7_SONNET'],
  ['claude-opus-4-1', 'VERTEX_REGION_CLAUDE_4_1_OPUS'],
  ['claude-opus-4', 'VERTEX_REGION_CLAUDE_4_0_OPUS'],
  ['claude-sonnet-4-6', 'VERTEX_REGION_CLAUDE_4_6_SONNET'],
  ['claude-sonnet-4-5', 'VERTEX_REGION_CLAUDE_4_5_SONNET'],
  ['claude-sonnet-4', 'VERTEX_REGION_CLAUDE_4_0_SONNET'],
]

export function getVertexRegionForModel(model: string | undefined): string {
  if (model) {
    const match = VERTEX_REGION_OVERRIDES.find(([prefix]) =>
      model.startsWith(prefix),
    )
    if (match) {
      return process.env[match[1]] || getDefaultVertexRegion()
    }
  }
  return getDefaultVertexRegion()
}

export function getDefaultVertexRegion(): string {
  return process.env.CLOUD_ML_REGION || 'us-east5'
}
```

**区域选择优先级:**

1. 模型特定环境变量 (`VERTEX_REGION_CLAUDE_*`)
2. 全局环境变量 (`CLOUD_ML_REGION`)
3. 默认区域 (`us-east5`)

### 4.3 GCP 凭证管理

```typescript
// utils/auth.ts:841-866
const GCP_CREDENTIALS_CHECK_TIMEOUT_MS = 5_000

export async function checkGcpCredentialsValid(): Promise<boolean> {
  try {
    const { GoogleAuth } = await import('google-auth-library')
    const auth = new GoogleAuth({
      scopes: ['https://www.googleapis.com/auth/cloud-platform'],
    })

    const probe = (async () => {
      const client = await auth.getClient()
      await client.getAccessToken()
    })()

    const timeout = sleep(GCP_CREDENTIALS_CHECK_TIMEOUT_MS).then(() => {
      throw new GcpCredentialsTimeoutError('GCP credentials check timed out')
    })

    await Promise.race([probe, timeout])
    return true
  } catch {
    return false
  }
}

// 凭证刷新
export const refreshGcpCredentialsIfNeeded = memoizeWithTTLAsync(
  async (): Promise<boolean> => {
    const refreshed = await runGcpAuthRefresh()
    return refreshed
  },
  DEFAULT_GCP_CREDENTIAL_TTL, // 1小时 TTL
)
```

**GCP Auth Refresh 执行:**

```typescript
// utils/auth.ts:875-967
const GCP_AUTH_REFRESH_TIMEOUT_MS = 3 * 60 * 1000

async function runGcpAuthRefresh(): Promise<boolean> {
  const gcpAuthRefresh = getConfiguredGcpAuthRefresh()
  if (!gcpAuthRefresh) return false

  // 安全检查: 项目配置需要 workspace trust
  if (isGcpAuthRefreshFromProjectSettings()) {
    const hasTrust = checkHasTrustDialogAccepted()
    if (!hasTrust) {
      logEvent('tengu_gcpAuthRefresh_missing_trust', {})
      return false
    }
  }

  // 检查凭证有效性
  const isValid = await checkGcpCredentialsValid()
  if (isValid) {
    logForDebugging('GCP credentials valid, skipping refresh')
    return false
  }

  return refreshGcpAuth(gcpAuthRefresh)
}

export function refreshGcpAuth(gcpAuthRefresh: string): Promise<boolean> {
  const authStatusManager = AwsAuthStatusManager.getInstance()
  authStatusManager.startAuthentication()

  return new Promise(resolve => {
    const refreshProc = exec(gcpAuthRefresh, {
      timeout: GCP_AUTH_REFRESH_TIMEOUT_MS,
    })

    refreshProc.stdout!.on('data', data => {
      authStatusManager.addOutput(data.toString().trim())
    })

    refreshProc.stderr!.on('data', data => {
      authStatusManager.setError(data.toString().trim())
    })

    refreshProc.on('close', (code, signal) => {
      if (code === 0) {
        authStatusManager.endAuthentication(true)
        resolve(true)
      } else {
        authStatusManager.endAuthentication(false)
        resolve(false)
      }
    })
  })
}
```

---

## 5. Azure Foundry

### 5.1 Foundry 客户端创建

```typescript
// services/api/client.ts:191-219
if (isEnvTruthy(process.env.CLAUDE_CODE_USE_FOUNDRY)) {
  const { AnthropicFoundry } = await import('@anthropic-ai/foundry-sdk')

  // Azure AD Token Provider 配置
  let azureADTokenProvider: (() => Promise<string>) | undefined

  if (!process.env.ANTHROPIC_FOUNDRY_API_KEY) {
    if (isEnvTruthy(process.env.CLAUDE_CODE_SKIP_FOUNDRY_AUTH)) {
      // 测试/代理场景: Mock token provider
      azureADTokenProvider = () => Promise.resolve('')
    } else {
      // 标准 Azure AD 认证
      const { DefaultAzureCredential, getBearerTokenProvider } =
        await import('@azure/identity')

      azureADTokenProvider = getBearerTokenProvider(
        new DefaultAzureCredential(),
        'https://cognitiveservices.azure.com/.default',
      )
    }
  }

  const foundryArgs = {
    ...ARGS,
    ...(azureADTokenProvider && { azureADTokenProvider }),
    ...(isDebugToStdErr() && { logger: createStderrLogger() }),
  }

  return new AnthropicFoundry(foundryArgs) as unknown as Anthropic
}
```

### 5.2 Foundry 认证方式

**两种认证路径:**

1. **API Key 认证:**
   ```bash
   export ANTHROPIC_FOUNDRY_API_KEY="your-api-key"
   ```

2. **Azure AD 认证 (DefaultAzureCredential):**
   ```typescript
   // 无 API key 时自动使用 Azure AD
   const azureADTokenProvider = getBearerTokenProvider(
     new DefaultAzureCredential(),
     'https://cognitiveservices.azure.com/.default',
   )
   ```

**DefaultAzureCredential 支持的认证方式:**

- 环境变量 (`AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`, `AZURE_TENANT_ID`)
- Managed Identity (Azure VM/Container Apps)
- Azure CLI (`az login`)
- Visual Studio Code
- Interactive Browser

### 5.3 Foundry 端点配置

```typescript
// 环境变量说明 (client.ts:43-54)
// ANTHROPIC_FOUNDRY_RESOURCE: Azure 资源名称
//   端点格式: https://{resource}.services.ai.azure.com/anthropic/v1/messages
//
// ANTHROPIC_FOUNDRY_BASE_URL: 完整 base URL (可选)
//   示例: 'https://my-resource.services.ai.azure.com'
```

---

## 6. 后端切换逻辑

### 6.1 主切换流程

```typescript
// services/api/client.ts:88-152
export async function getAnthropicClient({
  apiKey,
  maxRetries,
  model,
  fetchOverride,
  source,
}): Promise<Anthropic> {
  // 1. 构建通用 Headers
  const defaultHeaders = {
    'x-app': 'cli',
    'User-Agent': getUserAgent(),
    'X-Claude-Code-Session-Id': getSessionId(),
    ...customHeaders,
    ...(containerId && { 'x-claude-remote-container-id': containerId }),
    ...(remoteSessionId && { 'x-claude-remote-session-id': remoteSessionId }),
    ...(clientApp && { 'x-client-app': clientApp }),
  }

  // 2. OAuth token 检查
  await checkAndRefreshOAuthTokenIfNeeded()

  // 3. API key Headers 配置
  if (!isClaudeAISubscriber()) {
    await configureApiKeyHeaders(defaultHeaders, getIsNonInteractiveSession())
  }

  // 4. 构建通用参数
  const ARGS = {
    defaultHeaders,
    maxRetries,
    timeout: parseInt(process.env.API_TIMEOUT_MS || String(600 * 1000), 10),
    dangerouslyAllowBrowser: true,
    fetchOptions: getProxyFetchOptions({ forAnthropicAPI: true }),
    ...(resolvedFetch && { fetch: resolvedFetch }),
  }

  // 5. 按环境变量选择后端 (见各后端章节)
}
```

### 6.2 类型转换说明

```typescript
// 关键注释 (client.ts:188-189, 218-219, 296-297)
// "we have always been lying about the return type -
//  this doesn't support batching or models"

return new AnthropicBedrock(bedrockArgs) as unknown as Anthropic
return new AnthropicFoundry(foundryArgs) as unknown as Anthropic
return new AnthropicVertex(vertexArgs) as unknown as Anthropic
```

**设计原因:**

- 各 SDK (Bedrock/Vertex/Foundry) 类型不完全兼容
- 使用 `as unknown as Anthropic` 强制统一返回类型
- 核心 `messages.create` API 调用兼容
- `batching` 和 `models` API 在第三方后端不支持

### 6.3 重试逻辑适配

```typescript
// services/api/withRetry.ts:631-644
function isBedrockAuthError(error: unknown): boolean {
  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK)) {
    // AWS SDK 内部认证失败 (CredentialsProviderError)
    if (isAwsCredentialsProviderError(error)) return true
    // API 返回 403 "security token invalid"
    if (error instanceof APIError && error.status === 403) return true
  }
  return false
}

// services/api/withRetry.ts:670-682
function isVertexAuthError(error: unknown): boolean {
  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX)) {
    // google-auth-library 认证失败
    if (isGoogleAuthLibraryCredentialError(error)) return true
    // Vertex API 返回 401
    if (error instanceof APIError && error.status === 401) return true
  }
  return false
}

// 认证错误时清除缓存并重试
function handleAwsCredentialError(error: unknown): boolean {
  if (isBedrockAuthError(error)) {
    clearAwsCredentialsCache()
    return true
  }
  return false
}

function handleGcpCredentialError(error: unknown): boolean {
  if (isVertexAuthError(error)) {
    clearGcpCredentialsCache()
    return true
  }
  return false
}
```

---

## 7. 用户配置指南

### 7.1 Anthropic API 配置

**基础配置:**
```bash
# 方式1: 直接设置 API Key
export ANTHROPIC_API_KEY="sk-ant-..."

# 方式2: 使用 apiKeyHelper (动态获取)
# ~/.claude/settings.json
{
  "apiKeyHelper": "cat ~/.anthropic_api_key"
}

# 方式3: OAuth 登录 (交互式)
claude login
```

**高级配置:**
```bash
# 自定义 base URL (代理/网关)
export ANTHROPIC_BASE_URL="https://your-gateway.example.com"

# 自定义 headers
export ANTHROPIC_CUSTOM_HEADERS="X-Gateway-Key: abc123
Authorization: Bearer custom-token"

# Unix socket 代理 (SSH 远程)
export ANTHROPIC_UNIX_SOCKET="/tmp/claude-auth.sock"
```

### 7.2 AWS Bedrock 配置

**基础配置:**
```bash
# 启用 Bedrock
export CLAUDE_CODE_USE_BEDROCK="true"

# AWS 凭证 (标准链)
export AWS_REGION="us-east-1"
# 或
export AWS_DEFAULT_REGION="us-west-2"

# 凭证来源 (自动发现):
# 1. ~/.aws/credentials
# 2. ~/.aws/config
# 3. 环境变量 AWS_ACCESS_KEY_ID/AWS_SECRET_ACCESS_KEY
# 4. IAM role (EC2/Lambda)
```

**高级配置:**
```bash
# Bearer Token 认证 (Bedrock API Key)
export AWS_BEARER_TOKEN_BEDROCK="your-bedrock-api-key"

# 小模型区域覆盖
export ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION="eu-west-1"

# 自定义 Bedrock 端点
export ANTHROPIC_BEDROCK_BASE_URL="https://custom-bedrock.endpoint"

# 跳过认证 (代理场景)
export CLAUDE_CODE_SKIP_BEDROCK_AUTH="true"

# awsCredentialExport (动态凭证)
# ~/.claude/settings.json
{
  "awsCredentialExport": "aws sts assume-role --role-arn arn:aws:iam::123:role/MyRole --output json"
}

# awsAuthRefresh (自动刷新)
{
  "awsAuthRefresh": "aws sso login --profile my-profile"
}
```

### 7.3 Google Vertex AI 配置

**基础配置:**
```bash
# 启用 Vertex
export CLAUDE_CODE_USE_VERTEX="true"

# 项目 ID
export ANTHROPIC_VERTEX_PROJECT_ID="my-gcp-project"

# 区域配置
export CLOUD_ML_REGION="us-east5"

# 模型特定区域
export VERTEX_REGION_CLAUDE_OPUS_4_6="europe-west4"
export VERTEX_REGION_CLAUDE_SONNET_4_5="asia-east1"
```

**GCP 凭证来源:**
```bash
# 方式1: 服务账号 JSON
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/service-account.json"

# 方式2: gcloud ADC
gcloud auth application-default login

# 方式3: 环境变量
export GOOGLE_CLIENT_ID="..."
export GOOGLE_CLIENT_SECRET="..."
```

**高级配置:**
```bash
# 跳过认证
export CLAUDE_CODE_SKIP_VERTEX_AUTH="true"

# gcpAuthRefresh (自动刷新)
# ~/.claude/settings.json
{
  "gcpAuthRefresh": "gcloud auth application-default login"
}
```

### 7.4 Azure Foundry 配置

**基础配置:**
```bash
# 启用 Foundry
export CLAUDE_CODE_USE_FOUNDRY="true"

# Azure 资源名称
export ANTHROPIC_FOUNDRY_RESOURCE="my-azure-resource"

# API Key 认证
export ANTHROPIC_FOUNDRY_API_KEY="your-foundry-api-key"
```

**Azure AD 认证:**
```bash
# 方式1: 环境变量
export AZURE_CLIENT_ID="..."
export AZURE_CLIENT_SECRET="..."
export AZURE_TENANT_ID="..."

# 方式2: Azure CLI
az login

# 方式3: Managed Identity (自动)

# 方式4: 跳过认证 (代理)
export CLAUDE_CODE_SKIP_FOUNDRY_AUTH="true"
```

### 7.5 模型字符串配置

**模型 ID 映射表:**

```typescript
// utils/model/configs.ts
export const CLAUDE_OPUS_4_6_CONFIG = {
  firstParty: 'claude-opus-4-6',
  bedrock: 'us.anthropic.claude-opus-4-6-v1',
  vertex: 'claude-opus-4-6',
  foundry: 'claude-opus-4-6',
}

export const CLAUDE_SONNET_4_5_CONFIG = {
  firstParty: 'claude-sonnet-4-5-20250929',
  bedrock: 'us.anthropic.claude-sonnet-4-5-20250929-v1:0',
  vertex: 'claude-sonnet-4-5@20250929',
  foundry: 'claude-sonnet-4-5',
}
```

**模型字符串动态获取 (Bedrock):**
```typescript
// utils/model/modelStrings.ts:33-55
async function getBedrockModelStrings(): Promise<ModelStrings> {
  const profiles = await getBedrockInferenceProfiles()

  // 从 inference profile 列表匹配模型
  const out = {} as ModelStrings
  for (const key of MODEL_KEYS) {
    const needle = ALL_MODEL_CONFIGS[key].firstParty
    out[key] = findFirstMatch(profiles, needle) || fallback[key]
  }
  return out
}
```

**模型覆盖配置:**
```json
// ~/.claude/settings.json
{
  "modelOverrides": {
    "claude-opus-4-6": "arn:aws:bedrock:us-east-1:123:inference-profile/my-custom-profile"
  }
}
```

---

## 8. 从零实现

### 8.1 最小化实现

```typescript
// minimal-backend.ts
import Anthropic from '@anthropic-ai/sdk'

type BackendType = 'anthropic' | 'bedrock' | 'vertex' | 'foundry'

async function createClient(
  backend: BackendType,
  options: {
    apiKey?: string
    region?: string
    projectId?: string
  }
): Promise<Anthropic> {
  const commonConfig = {
    maxRetries: 3,
    timeout: 600000,
  }

  switch (backend) {
    case 'anthropic':
      return new Anthropic({
        ...commonConfig,
        apiKey: options.apiKey,
      })

    case 'bedrock':
      const { AnthropicBedrock } = await import('@anthropic-ai/bedrock-sdk')
      return new AnthropicBedrock({
        ...commonConfig,
        awsRegion: options.region || 'us-east-1',
      }) as unknown as Anthropic

    case 'vertex':
      const [{ AnthropicVertex }, { GoogleAuth }] = await Promise.all([
        import('@anthropic-ai/vertex-sdk'),
        import('google-auth-library'),
      ])
      return new AnthropicVertex({
        ...commonConfig,
        region: options.region || 'us-east5',
        googleAuth: new GoogleAuth({
          projectId: options.projectId,
        }),
      }) as unknown as Anthropic

    case 'foundry':
      const { AnthropicFoundry } = await import('@anthropic-ai/foundry-sdk')
      const { DefaultAzureCredential, getBearerTokenProvider } =
        await import('@azure/identity')
      return new AnthropicFoundry({
        ...commonConfig,
        azureADTokenProvider: getBearerTokenProvider(
          new DefaultAzureCredential(),
          'https://cognitiveservices.azure.com/.default',
        ),
      }) as unknown as Anthropic
  }
}
```

### 8.2 环境变量驱动实现

```typescript
// env-driven-backend.ts
import { isEnvTruthy } from './envUtils.js'

export type APIProvider = 'firstParty' | 'bedrock' | 'vertex' | 'foundry'

export function getAPIProvider(): APIProvider {
  return isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK)
    ? 'bedrock'
    : isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX)
      ? 'vertex'
      : isEnvTruthy(process.env.CLAUDE_CODE_USE_FOUNDRY)
        ? 'foundry'
        : 'firstParty'
}

export function isEnvTruthy(value: string | undefined): boolean {
  if (!value) return false
  const normalized = value.toLowerCase().trim()
  return ['1', 'true', 'yes', 'on'].includes(normalized)
}

export function getAWSRegion(): string {
  return process.env.AWS_REGION ||
         process.env.AWS_DEFAULT_REGION ||
         'us-east-1'
}

export function getVertexRegion(): string {
  return process.env.CLOUD_ML_REGION || 'us-east5'
}
```

### 8.3 认证管理实现

```typescript
// auth-manager.ts
import memoize from 'lodash-es/memoize.js'

// AWS 凭证管理
export const refreshAwsCredentials = memoize(
  async (): Promise<{
    accessKeyId: string
    secretAccessKey: string
    sessionToken: string
  } | null> => {
    try {
      const { STSClient, GetCallerIdentityCommand } =
        await import('@aws-sdk/client-sts')
      await new STSClient().send(new GetCallerIdentityCommand({}))

      // 已有有效凭证
      return null
    } catch {
      // 需要刷新凭证
      const { fromIni } = await import('@aws-sdk/credential-providers')
      const creds = await fromIni()()
      return {
        accessKeyId: creds.accessKeyId,
        secretAccessKey: creds.secretAccessKey,
        sessionToken: creds.sessionToken,
      }
    }
  },
  { maxAge: 5 * 60 * 1000 } // 5分钟缓存
)

// GCP 凭证管理
export const refreshGcpCredentials = memoize(
  async (): Promise<boolean> => {
    try {
      const { GoogleAuth } = await import('google-auth-library')
      const auth = new GoogleAuth({
        scopes: ['https://www.googleapis.com/auth/cloud-platform'],
      })
      const client = await auth.getClient()
      await client.getAccessToken()
      return true
    } catch {
      return false
    }
  },
  { maxAge: 60 * 60 * 1000 } // 1小时缓存
)
```

### 8.4 模型配置实现

```typescript
// model-configs.ts
type ModelConfig = Record<APIProvider, string>

export const MODEL_CONFIGS: Record<string, ModelConfig> = {
  opus46: {
    firstParty: 'claude-opus-4-6',
    bedrock: 'us.anthropic.claude-opus-4-6-v1',
    vertex: 'claude-opus-4-6',
    foundry: 'claude-opus-4-6',
  },
  sonnet45: {
    firstParty: 'claude-sonnet-4-5-20250929',
    bedrock: 'us.anthropic.claude-sonnet-4-5-20250929-v1:0',
    vertex: 'claude-sonnet-4-5@20250929',
    foundry: 'claude-sonnet-4-5',
  },
  haiku45: {
    firstParty: 'claude-haiku-4-5-20251001',
    bedrock: 'us.anthropic.claude-haiku-4-5-20251001-v1:0',
    vertex: 'claude-haiku-4-5@20251001',
    foundry: 'claude-haiku-4-5',
  },
}

export function getModelId(
  modelKey: string,
  provider: APIProvider
): string {
  return MODEL_CONFIGS[modelKey]?.[provider] || MODEL_CONFIGS[modelKey].firstParty
}

export function getDefaultModel(provider: APIProvider): string {
  // 第三方后端默认使用较稳定的模型版本
  if (provider !== 'firstParty') {
    return getModelId('sonnet45', provider)
  }
  return getModelId('sonnet46', provider)
}
```

### 8.5 重试逻辑实现

```typescript
// retry-handler.ts
import { APIError, APIConnectionError } from '@anthropic-ai/sdk'

const MAX_RETRIES = 10
const BASE_DELAY_MS = 500

export function shouldRetry(error: unknown): boolean {
  if (!(error instanceof APIError)) return false

  // 连接错误
  if (error instanceof APIConnectionError) return true

  // 请求超时
  if (error.status === 408) return true

  // 锁超时
  if (error.status === 409) return true

  // 服务端错误
  if (error.status >= 500) return true

  // 认证错误 (清除缓存后重试)
  if (error.status === 401) return true

  return false
}

export function getRetryDelay(attempt: number): number {
  const baseDelay = Math.min(
    BASE_DELAY_MS * Math.pow(2, attempt - 1),
    32000
  )
  const jitter = Math.random() * 0.25 * baseDelay
  return baseDelay + jitter
}

export async function withRetry<T>(
  operation: () => Promise<T>,
  options: { maxRetries?: number } = {}
): Promise<T> {
  const maxRetries = options.maxRetries ?? MAX_RETRIES

  for (let attempt = 1; attempt <= maxRetries + 1; attempt++) {
    try {
      return await operation()
    } catch (error) {
      if (attempt > maxRetries || !shouldRetry(error)) {
        throw error
      }

      const delay = getRetryDelay(attempt)
      console.log(`Retry ${attempt}/${maxRetries} after ${delay}ms`)
      await new Promise(resolve => setTimeout(resolve, delay))
    }
  }

  throw new Error('Max retries exceeded')
}
```

### 8.6 代理支持实现

```typescript
// proxy-support.ts
import { HttpsProxyAgent } from 'https-proxy-agent'

export function getProxyUrl(): string | undefined {
  return process.env.https_proxy ||
         process.env.HTTPS_PROXY ||
         process.env.http_proxy ||
         process.env.HTTP_PROXY
}

export function getNoProxy(): string | undefined {
  return process.env.no_proxy || process.env.NO_PROXY
}

export function shouldBypassProxy(url: string): boolean {
  const noProxy = getNoProxy()
  if (!noProxy) return false
  if (noProxy === '*') return true

  const hostname = new URL(url).hostname.toLowerCase()
  const patterns = noProxy.split(/[,\s]+/)

  return patterns.some(pattern => {
    pattern = pattern.toLowerCase().trim()
    if (pattern.startsWith('.')) {
      return hostname.endsWith(pattern) ||
             hostname === pattern.substring(1)
    }
    return hostname === pattern
  })
}

export function createProxyAgent(): HttpsProxyAgent | undefined {
  const proxyUrl = getProxyUrl()
  if (!proxyUrl) return undefined

  return new HttpsProxyAgent(proxyUrl)
}
```

---

## 附录: 文件路径索引

| 功能模块 | 文件路径 |
|---------|---------|
| API 客户端工厂 | `services/api/client.ts` |
| 后端类型定义 | `utils/model/providers.ts` |
| AWS 凭证管理 | `utils/aws.ts` |
| AWS 凭证刷新 | `utils/auth.ts:701-807` |
| GCP 凭证刷新 | `utils/auth.ts:841-985` |
| Bedrock 模型配置 | `utils/model/bedrock.ts` |
| 模型字符串映射 | `utils/model/modelStrings.ts` |
| 模型 ID 配置 | `utils/model/configs.ts` |
| 环境变量工具 | `utils/envUtils.ts` |
| 代理配置 | `utils/proxy.ts` |
| 重试逻辑 | `services/api/withRetry.ts` |
| API 预连接 | `utils/apiPreconnect.ts` |

---

*分析完成: 2026-04-02*