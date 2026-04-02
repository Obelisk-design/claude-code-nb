# MCP服务器开发指南

**生成日期:** 2026-04-02  
**分析范围:** `services/mcp/`, `commands/mcp/`, `tools/MCPTool/`, `entrypoints/mcp.ts`

---

## 1. MCP协议深入

### 1.1 协议架构

MCP (Model Context Protocol) 是Anthropic设计的标准化协议，用于Claude Code与外部工具/服务的通信。

**核心组件：**
- **SDK包:** `@modelcontextprotocol/sdk`
- **传输层:** 多种传输机制（stdio、SSE、HTTP、WebSocket）
- **消息格式:** JSON-RPC 2.0规范

**协议层次：**

```
┌─────────────────────────────────────┐
│  Claude Code CLI (MCP Client)       │
│  services/mcp/client.ts             │
├─────────────────────────────────────┤
│  MCP SDK Client                     │
│  @modelcontextprotocol/sdk/client   │
├─────────────────────────────────────┤
│  Transport Layer                    │
│  stdio | SSE | HTTP | WebSocket     │
├─────────────────────────────────────┤
│  MCP Server (外部服务)               │
│  @modelcontextprotocol/sdk/server   │
└─────────────────────────────────────┘
```

### 1.2 消息类型

**核心消息类型（来自SDK）：**

```typescript
// 来自 @modelcontextprotocol/sdk/types.js
import {
  ListToolsRequestSchema,
  CallToolRequestSchema,
  ListResourcesRequestSchema,
  ReadResourceRequestSchema,
  ListPromptsRequestSchema,
  GetPromptRequestSchema,
} from '@modelcontextprotocol/sdk/types.js'
```

**消息流示例：**

```
Client → Server: ListToolsRequest
Server → Client: ListToolsResult (返回工具列表)

Client → Server: CallToolRequest (name, arguments)
Server → Client: CallToolResult (content, isError)
```

### 1.3 能力声明

**服务器能力定义：**

```typescript
// 来自 entrypoints/mcp.ts
const server = new Server(
  {
    name: 'claude/tengu',
    version: MACRO.VERSION,
  },
  {
    capabilities: {
      tools: {},      // 支持工具调用
      resources: {},  // 支持资源访问
      prompts: {},    // 支持提示模板
    },
  },
)
```

---

## 2. 服务器配置

### 2.1 配置类型系统

**核心配置类型（`services/mcp/types.ts`）：**

```typescript
// 传输类型枚举
export const TransportSchema = lazySchema(() =>
  z.enum(['stdio', 'sse', 'sse-ide', 'http', 'ws', 'sdk']),
)

// 配置作用域
export const ConfigScopeSchema = lazySchema(() =>
  z.enum([
    'local',      // 本地配置（仅当前项目可见）
    'user',       // 用户配置（全局可用）
    'project',    // 项目配置（通过.mcp.json共享）
    'dynamic',    // 动态配置（命令行指定）
    'enterprise', // 企业配置（托管）
    'claudeai',   // Claude.ai连接器
    'managed',    // 管理配置
  ]),
)
```

### 2.2 Stdio服务器配置

**最常用的本地服务器类型：**

```typescript
// 来自 services/mcp/types.ts:28-35
export const McpStdioServerConfigSchema = lazySchema(() =>
  z.object({
    type: z.literal('stdio').optional(),
    command: z.string().min(1, 'Command cannot be empty'),
    args: z.array(z.string()).default([]),
    env: z.record(z.string(), z.string()).optional(),
  }),
)
```

**配置示例：**

```json
{
  "mcpServers": {
    "my-server": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "my-mcp-server"],
      "env": {
        "API_KEY": "xxx"
      }
    }
  }
}
```

### 2.3 HTTP/SSE服务器配置

**远程服务器配置：**

```typescript
// SSE服务器配置
export const McpSSEServerConfigSchema = lazySchema(() =>
  z.object({
    type: z.literal('sse'),
    url: z.string(),
    headers: z.record(z.string(), z.string()).optional(),
    headersHelper: z.string().optional(),
    oauth: McpOAuthConfigSchema().optional(),
  }),
)

// HTTP服务器配置（Streamable HTTP）
export const McpHTTPServerConfigSchema = lazySchema(() =>
  z.object({
    type: z.literal('http'),
    url: z.string(),
    headers: z.record(z.string(), z.string()).optional(),
    headersHelper: z.string().optional(),
    oauth: McpOAuthConfigSchema().optional(),
  }),
)
```

### 2.4 OAuth配置

**OAuth认证配置：**

```typescript
// 来自 services/mcp/types.ts:43-56
const McpOAuthConfigSchema = lazySchema(() =>
  z.object({
    clientId: z.string().optional(),
    callbackPort: z.number().int().positive().optional(),
    authServerMetadataUrl: z.string()
      .url()
      .startsWith('https://', {
        message: 'authServerMetadataUrl must use https://',
      })
      .optional(),
    xaa: McpXaaConfigSchema().optional(),  // Cross-App Access
  }),
)
```

### 2.5 配置加载流程

**配置加载层次（`services/mcp/config.ts`）：**

```typescript
// 配置合并优先级（从低到高）
1. enterprise (托管配置)
2. claudeai (Claude.ai连接器)
3. project (.mcp.json)
4. user (全局settings.json)
5. local (项目settings.json)
6. dynamic (命令行)
```

**配置加载核心函数：**

```typescript
// getAllMcpConfigs() 合并所有作用域的配置
export async function getAllMcpConfigs(): Promise<
  Record<string, ScopedMcpServerConfig>
> {
  // 1. 加载托管配置（企业）
  const enterpriseConfigs = await loadEnterpriseMcpConfig()
  
  // 2. 加载Claude.ai连接器
  const claudeAiConfigs = await fetchClaudeAIMcpConfigsIfEligible()
  
  // 3. 加载项目.mcp.json
  const projectConfigs = await loadProjectMcpConfig()
  
  // 4. 加载用户配置
  const userConfigs = loadUserMcpConfig()
  
  // 5. 加载本地配置
  const localConfigs = loadLocalMcpConfig()
  
  // 6. 合并配置（去重处理）
  return mergeConfigs({
    enterprise: enterpriseConfigs,
    claudeai: claudeAiConfigs,
    project: projectConfigs,
    user: userConfigs,
    local: localConfigs,
  })
}
```

### 2.6 配置文件位置

**各作用域的配置文件路径：**

```typescript
// 来自 services/mcp/utils.ts:263-280
export function describeMcpConfigFilePath(scope: ConfigScope): string {
  switch (scope) {
    case 'user':
      return getGlobalClaudeFile()  // ~/.claude/settings.json
    case 'project':
      return join(getCwd(), '.mcp.json')  // 项目根目录
    case 'local':
      return `${getGlobalClaudeFile()} [project: ${getCwd()}]`
    case 'dynamic':
      return 'Dynamically configured'
    case 'enterprise':
      return getEnterpriseMcpFilePath()  // 托管配置路径
    case 'claudeai':
      return 'claude.ai'
    default:
      return scope
  }
}
```

---

## 3. 工具定义

### 3.1 工具系统架构

**MCP工具命名规范：**

```typescript
// 来自 services/mcp/mcpStringUtils.ts:19-32
export function mcpInfoFromString(toolString: string): {
  serverName: string
  toolName: string | undefined
} | null {
  const parts = toolString.split('__')
  const [mcpPart, serverName, ...toolNameParts] = parts
  
  // 格式: mcp__serverName__toolName
  if (mcpPart !== 'mcp' || !serverName) {
    return null
  }
  
  const toolName = toolNameParts.length > 0 
    ? toolNameParts.join('__') 
    : undefined
    
  return { serverName, toolName }
}
```

**构建工具名称：**

```typescript
// 来自 services/mcp/mcpStringUtils.ts:50-52
export function buildMcpToolName(serverName: string, toolName: string): string {
  return `${getMcpPrefix(serverName)}${normalizeNameForMCP(toolName)}`
}

// 前缀生成
export function getMcpPrefix(serverName: string): string {
  return `mcp__${normalizeNameForMCP(serverName)}__`
}
```

### 3.2 名称规范化

**规范化规则（`services/mcp/normalization.ts`）：**

```typescript
// 来自 services/mcp/normalization.ts:17-23
export function normalizeNameForMCP(name: string): string {
  // 替换非法字符为下划线（只允许 a-zA-Z0-9_-）
  let normalized = name.replace(/[^a-zA-Z0-9_-]/g, '_')
  
  // Claude.ai服务器特殊处理：压缩连续下划线
  if (name.startsWith('claude.ai ')) {
    normalized = normalized
      .replace(/_+/g, '_')
      .replace(/^_|_$/g, '')
  }
  
  return normalized
}
```

### 3.3 MCPTool实现

**基础工具定义（`tools/MCPTool/MCPTool.ts`）：**

```typescript
// 来自 tools/MCPTool/MCPTool.ts:27-77
export const MCPTool = buildTool({
  isMcp: true,
  name: 'mcp',  // 基础名称，实际调用时会被覆盖
  maxResultSizeChars: 100_000,
  
  // 输入Schema：允许任意对象（MCP工具自定义Schema）
  get inputSchema(): InputSchema {
    return inputSchema()  // z.object({}).passthrough()
  },
  
  // 输出Schema：字符串结果
  get outputSchema(): OutputSchema {
    return outputSchema()  // z.string()
  },
  
  // 权限检查
  async checkPermissions(): Promise<PermissionResult> {
    return {
      behavior: 'passthrough',
      message: 'MCPTool requires permission.',
    }
  },
  
  // 渲染函数
  renderToolUseMessage,
  renderToolUseProgressMessage,
  renderToolResultMessage,
  
  // 结果截断检测
  isResultTruncated(output: Output): boolean {
    return isOutputLineTruncated(output)
  },
})
```

### 3.4 工具调用流程

**工具调用核心流程（`services/mcp/client.ts`）：**

```typescript
// 工具调用处理
async function callMcpTool(
  client: Client,
  toolName: string,
  args: Record<string, unknown>,
): Promise<CallToolResult> {
  // 1. 发送调用请求
  const result = await client.request(
    {
      method: 'tools/call',
      params: {
        name: toolName,
        arguments: args,
      },
    },
    CallToolResultSchema,
  )
  
  // 2. 处理结果
  if (result.isError) {
    throw new McpToolCallError(
      'Tool execution failed',
      'MCP tool returned error',
      { _meta: result._meta },
    )
  }
  
  // 3. 转换内容格式
  return processToolResult(result.content)
}
```

### 3.5 工具列表获取

**获取服务器工具列表：**

```typescript
// 来自 services/mcp/client.ts
export async function fetchToolsForClient(
  client: Client,
  serverName: string,
): Promise<Tool[]> {
  // 发送ListTools请求
  const result = await client.request(
    { method: 'tools/list' },
    ListToolsResultSchema,
  )
  
  // 转换为Claude Code工具格式
  return result.tools.map(tool => ({
    name: buildMcpToolName(serverName, tool.name),
    description: tool.description,
    inputSchema: tool.inputSchema,
    outputSchema: tool.outputSchema,
    isMcp: true,
    mcpInfo: {
      serverName,
      toolName: tool.name,
    },
  }))
}
```

---

## 4. 资源暴露

### 4.1 资源类型定义

**资源结构（`services/mcp/types.ts`）：**

```typescript
// 资源类型（来自SDK）
import { Resource } from '@modelcontextprotocol/sdk/types.js'

// 扩展资源类型（带服务器标识）
export type ServerResource = Resource & { server: string }

// MCP CLI状态中的资源存储
export interface MCPCliState {
  clients: SerializedClient[]
  configs: Record<string, ScopedMcpServerConfig>
  tools: SerializedTool[]
  resources: Record<string, ServerResource[]>  // 按服务器分组
}
```

**SDK Resource类型：**

```typescript
// 来自 @modelcontextprotocol/sdk/types.js
interface Resource {
  uri: string           // 资源URI
  name: string          // 资源名称
  description?: string  // 资源描述
  mimeType?: string     // MIME类型
}
```

### 4.2 资源访问流程

**获取资源列表：**

```typescript
// 来自 services/mcp/client.ts
export async function fetchResourcesForClient(
  client: Client,
  serverName: string,
): Promise<ServerResource[]> {
  // 发送ListResources请求
  const result = await client.request(
    { method: 'resources/list' },
    ListResourcesResultSchema,
  )
  
  // 添加服务器标识
  return result.resources.map(resource => ({
    ...resource,
    server: serverName,
  }))
}
```

**读取资源内容：**

```typescript
// 资源读取工具 (ReadMcpResourceTool)
async function readMcpResource(
  client: Client,
  uri: string,
): Promise<ResourceContents> {
  const result = await client.request(
    {
      method: 'resources/read',
      params: { uri },
    },
    ReadResourceResultSchema,
  )
  
  return result.contents
}
```

### 4.3 资源过滤与管理

**按服务器过滤资源：**

```typescript
// 来自 services/mcp/utils.ts:102-107
export function filterResourcesByServer(
  resources: ServerResource[],
  serverName: string,
): ServerResource[] {
  return resources.filter(resource => resource.server === serverName)
}

// 排除特定服务器资源
export function excludeResourcesByServer(
  resources: Record<string, ServerResource[]>,
  serverName: string,
): Record<string, ServerResource[]> {
  const result = { ...resources }
  delete result[serverName]
  return result
}
```

---

## 5. 认证集成

### 5.1 OAuth认证流程

**ClaudeAuthProvider实现（`services/mcp/auth.ts`）：**

```typescript
// 来自 services/mcp/auth.ts
export class ClaudeAuthProvider implements OAuthClientProvider {
  private serverName: string
  private serverConfig: McpSSEServerConfig | McpHTTPServerConfig
  
  // 获取客户端信息
  async clientInformation(): Promise<OAuthClientInformation | undefined> {
    // 从安全存储读取clientId/clientSecret
    return {
      client_id: await readClientId(this.serverName),
      client_secret: await readClientSecret(this.serverName),
    }
  }
  
  // 获取令牌
  async tokens(): Promise<OAuthTokens | undefined> {
    // 从keychain读取存储的令牌
    return await readOAuthTokens(this.serverName)
  }
  
  // 保存储牌
  async saveTokens(tokens: OAuthTokens): Promise<void> {
    await saveOAuthTokens(this.serverName, tokens)
  }
  
  // 执行OAuth授权流程
  async authorize(): Promise<void> {
    // 1. 发现授权服务器元数据
    const metadata = await fetchAuthServerMetadata(...)
    
    // 2. 构建授权URL
    const authUrl = buildAuthorizationUrl(metadata)
    
    // 3. 打开浏览器授权
    await openBrowser(authUrl)
    
    // 4. 等待回调接收code
    const code = await waitForCallback()
    
    // 5. 交换令牌
    const tokens = await exchangeCodeForTokens(code)
    
    // 6. 保存储牌
    await this.saveTokens(tokens)
  }
}
```

### 5.2 OAuth元数据发现

**RFC 9728 → RFC 8414发现流程：**

```typescript
// 来自 services/mcp/auth.ts:255-299
async function fetchAuthServerMetadata(
  serverName: string,
  serverUrl: string,
  configuredMetadataUrl: string | undefined,
  fetchFn?: FetchLike,
  resourceMetadataUrl?: URL,
): Promise<AuthorizationServerMetadata> {
  // 1. 优先使用配置的元数据URL
  if (configuredMetadataUrl) {
    if (!configuredMetadataUrl.startsWith('https://')) {
      throw new Error('authServerMetadataUrl must use https://')
    }
    const response = await authFetch(configuredMetadataUrl)
    return OAuthMetadataSchema.parse(await response.json())
  }
  
  // 2. RFC 9728发现（MCP服务器元数据）
  try {
    const { authorizationServerMetadata } = await discoverOAuthServerInfo(
      serverUrl,
      { fetchFn, resourceMetadataUrl },
    )
    if (authorizationServerMetadata) {
      return authorizationServerMetadata
    }
  } catch (err) {
    logMCPDebug(serverName, 'RFC 9728 discovery failed, falling back')
  }
  
  // 3. RFC 8414 fallback（传统OAuth发现）
  return await discoverAuthorizationServerMetadata(serverUrl, { fetchFn })
}
```

### 5.3 令牌刷新机制

**自动刷新流程：**

```typescript
// 来自 services/mcp/auth.ts
async function refreshOAuthToken(
  serverName: string,
  serverConfig: McpSSEServerConfig | McpHTTPServerConfig,
): Promise<OAuthTokens> {
  // 1. 获取当前令牌
  const currentTokens = await readOAuthTokens(serverName)
  if (!currentTokens?.refresh_token) {
    throw new Error('No refresh token available')
  }
  
  // 2. 发现授权服务器
  const metadata = await fetchAuthServerMetadata(...)
  
  // 3. 构建刷新请求
  const refreshUrl = metadata.token_endpoint
  const refreshBody = {
    grant_type: 'refresh_token',
    refresh_token: currentTokens.refresh_token,
  }
  
  // 4. 发送刷新请求
  const response = await authFetch(refreshUrl, {
    method: 'POST',
    body: jsonStringify(refreshBody),
  })
  
  // 5. 解析新令牌
  const newTokens = OAuthTokensSchema.parse(await response.json())
  
  // 6. 保存储牌
  await saveOAuthTokens(serverName, newTokens)
  
  return newTokens
}
```

### 5.4 XAA (Cross-App Access)

**跨应用访问机制：**

```typescript
// 来自 services/mcp/xaa.ts
export async function performCrossAppAccess(
  serverName: string,
  serverConfig: ScopedMcpServerConfig,
): Promise<string> {
  // 1. 获取IdP令牌
  const idpToken = await acquireIdpIdToken()
  
  // 2. 执行XAA令牌交换
  const xaaToken = await exchangeXaaToken(idpToken, serverConfig)
  
  // 3. 返回访问令牌
  return xaaToken.access_token
}
```

**XAA配置要求：**

```typescript
// 来自 services/mcp/xaaIdpLogin.ts
export function isXaaEnabled(): boolean {
  const settings = getXaaIdpSettings()
  return settings !== undefined && process.env.CLAUDE_CODE_ENABLE_XAA === '1'
}

// IdP设置结构
interface XaaIdpSettings {
  issuer: string      // OIDC提供商URL
  clientId: string    // 客户端ID
  callbackPort: number // 回调端口
}
```

---

## 6. 开发最佳实践

### 6.1 错误处理

**错误分类与处理：**

```typescript
// 来自 services/mcp/client.ts:151-186
// 认证错误
export class McpAuthError extends Error {
  serverName: string
  constructor(serverName: string, message: string) {
    super(message)
    this.name = 'McpAuthError'
    this.serverName = serverName
  }
}

// 会话过期错误
class McpSessionExpiredError extends Error {
  constructor(serverName: string) {
    super(`MCP server "${serverName}" session expired`)
    this.name = 'McpSessionExpiredError'
  }
}

// 工具调用错误
export class McpToolCallError extends TelemetrySafeError {
  constructor(
    message: string,
    telemetryMessage: string,
    readonly mcpMeta?: { _meta?: Record<string, unknown> },
  ) {
    super(message, telemetryMessage)
    this.name = 'McpToolCallError'
  }
}
```

**错误检测函数：**

```typescript
// 来自 services/mcp/client.ts:192-205
export function isMcpSessionExpiredError(error: Error): boolean {
  // 检查HTTP状态码
  const httpStatus = 'code' in error 
    ? (error as Error & { code?: number }).code 
    : undefined
  
  if (httpStatus !== 404) {
    return false
  }
  
  // 检查JSON-RPC错误码（-32001 = Session not found）
  return (
    error.message.includes('"code":-32001') ||
    error.message.includes('"code": -32001')
  )
}
```

### 6.2 连接管理

**连接状态类型：**

```typescript
// 来自 services/mcp/types.ts:180-227
export type MCPServerConnection =
  | ConnectedMCPServer    // 已连接
  | FailedMCPServer       // 连接失败
  | NeedsAuthMCPServer    // 需要认证
  | PendingMCPServer      // 等待连接
  | DisabledMCPServer     // 已禁用

// 已连接服务器
type ConnectedMCPServer = {
  client: Client
  name: string
  type: 'connected'
  capabilities: ServerCapabilities
  serverInfo?: { name: string; version: string }
  instructions?: string
  config: ScopedMcpServerConfig
  cleanup: () => Promise<void>
}
```

**连接缓存管理：**

```typescript
// 来自 services/mcp/client.ts:581-586
export function getServerCacheKey(
  name: string,
  serverRef: ScopedMcpServerConfig,
): string {
  // 使用名称+配置作为缓存键
  return `${name}-${jsonStringify(serverRef)}`
}

// 清除服务器缓存
export function clearServerCache(name: string, serverRef: ScopedMcpServerConfig): void {
  const cacheKey = getServerCacheKey(name, serverRef)
  connectToServer.cache.delete(cacheKey)
}
```

### 6.3 重连机制

**指数退避重连：**

```typescript
// 来自 services/mcp/useManageMCPConnections.ts:86-89
const MAX_RECONNECT_ATTEMPTS = 5
const INITIAL_BACKOFF_MS = 1000
const MAX_BACKOFF_MS = 30000

async function reconnectWithBackoff(
  serverName: string,
  serverConfig: ScopedMcpServerConfig,
): Promise<void> {
  for (let attempt = 0; attempt < MAX_RECONNECT_ATTEMPTS; attempt++) {
    try {
      await connectToServer(serverName, serverConfig)
      return  // 成功连接
    } catch (error) {
      const backoff = Math.min(
        INITIAL_BACKOFF_MS * Math.pow(2, attempt),
        MAX_BACKOFF_MS,
      )
      await sleep(backoff)
    }
  }
  throw new Error('Max reconnect attempts exhausted')
}
```

### 6.4 资源清理

**清理注册机制：**

```typescript
// 注册清理函数
export function registerCleanup(cleanup: () => Promise<void>): void {
  // 在进程退出时执行清理
  process.on('exit', async () => {
    await cleanup()
  })
}

// 服务器清理函数
async function cleanupServer(client: ConnectedMCPServer): Promise<void> {
  // 1. 关闭客户端连接
  await client.client.close()
  
  // 2. 清理工具/命令/资源
  excludeToolsByServer(tools, client.name)
  excludeCommandsByServer(commands, client.name)
  excludeResourcesByServer(resources, client.name)
  
  // 3. 清除缓存
  clearServerCache(client.name, client.config)
}
```

### 6.5 超时管理

**请求超时配置：**

```typescript
// 来自 services/mcp/client.ts:210-228
// 默认工具调用超时（约27.8小时）
const DEFAULT_MCP_TOOL_TIMEOUT_MS = 100_000_000

// 单个请求超时（60秒）
const MCP_REQUEST_TIMEOUT_MS = 60000

// 认证请求超时（30秒）
const AUTH_REQUEST_TIMEOUT_MS = 30000

function getMcpToolTimeoutMs(): number {
  return parseInt(process.env.MCP_TOOL_TIMEOUT || '', 10) ||
    DEFAULT_MCP_TOOL_TIMEOUT_MS
}

// 包装fetch以应用超时
export function wrapFetchWithTimeout(baseFetch: FetchLike): FetchLike {
  return async (url: string | URL, init?: RequestInit) => {
    const method = (init?.method ?? 'GET').toUpperCase()
    
    // GET请求不应用超时（长连接SSE流）
    if (method === 'GET') {
      return baseFetch(url, init)
    }
    
    // POST请求应用60秒超时
    const controller = new AbortController()
    const timer = setTimeout(
      () => controller.abort(new DOMException('Timeout', 'TimeoutError')),
      MCP_REQUEST_TIMEOUT_MS,
    )
    
    try {
      const response = await baseFetch(url, {
        ...init,
        signal: controller.signal,
      })
      clearTimeout(timer)
      return response
    } catch (error) {
      clearTimeout(timer)
      throw error
    }
  }
}
```

### 6.6 并发控制

**批量连接控制：**

```typescript
// 来自 services/mcp/client.ts:552-561
export function getMcpServerConnectionBatchSize(): number {
  return parseInt(process.env.MCP_SERVER_CONNECTION_BATCH_SIZE || '', 10) || 3
}

function getRemoteMcpServerConnectionBatchSize(): number {
  return parseInt(process.env.MCP_REMOTE_SERVER_CONNECTION_BATCH_SIZE || '', 10) || 20
}

// 使用pMap控制并发
import pMap from 'p-map'

async function connectToServers(servers: ServerConfig[]): Promise<MCPServerConnection[]> {
  return pMap(servers, connectToServer, {
    concurrency: getMcpServerConnectionBatchSize(),
  })
}
```

---

## 7. 完整示例

### 7.1 创建Stdio MCP服务器

**最小化MCP服务器实现：**

```typescript
// my-mcp-server.ts
import { Server } from '@modelcontextprotocol/sdk/server/index.js'
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js'
import {
  ListToolsRequestSchema,
  CallToolRequestSchema,
} from '@modelcontextprotocol/sdk/types.js'

// 创建服务器实例
const server = new Server(
  {
    name: 'my-mcp-server',
    version: '1.0.0',
  },
  {
    capabilities: {
      tools: {},
    },
  },
)

// 注册工具列表处理器
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: 'echo',
        description: 'Echo the input message',
        inputSchema: {
          type: 'object',
          properties: {
            message: {
              type: 'string',
              description: 'Message to echo',
            },
          },
          required: ['message'],
        },
      },
    ],
  }
})

// 注册工具调用处理器
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params
  
  if (name === 'echo') {
    return {
      content: [
        {
          type: 'text',
          text: args.message,
        },
      ],
    }
  }
  
  throw new Error(`Unknown tool: ${name}`)
})

// 启动服务器
async function run() {
  const transport = new StdioServerTransport()
  await server.connect(transport)
}

run()
```

### 7.2 创建HTTP MCP服务器

**Streamable HTTP服务器实现：**

```typescript
// my-http-mcp-server.ts
import { Server } from '@modelcontextprotocol/sdk/server/index.js'
import { StreamableHTTPServerTransport } from '@modelcontextprotocol/sdk/server/streamableHttp.js'
import express from 'express'

const server = new Server(
  { name: 'my-http-server', version: '1.0.0' },
  { capabilities: { tools: {} } },
)

// 注册处理器（同stdio示例）
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: 'fetch_url',
        description: 'Fetch content from URL',
        inputSchema: {
          type: 'object',
          properties: {
            url: { type: 'string', description: 'URL to fetch' },
          },
          required: ['url'],
        },
      },
    ],
  }
})

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params
  
  if (name === 'fetch_url') {
    const response = await fetch(args.url)
    const content = await response.text()
    return {
      content: [{ type: 'text', text: content }],
    }
  }
  
  throw new Error(`Unknown tool: ${name}`)
})

// Express服务器
const app = express()
const transport = new StreamableHTTPServerTransport()

// MCP endpoint
app.post('/mcp', async (req, res) => {
  await transport.handleRequest(req, res)
})

app.listen(3000, () => {
  console.log('MCP server running on http://localhost:3000/mcp')
  server.connect(transport)
})
```

### 7.3 配置与添加服务器

**使用CLI添加服务器：**

```bash
# Stdio服务器
claude mcp add my-server -- npx -y my-mcp-server

# HTTP服务器
claude mcp add --transport http my-http-server https://localhost:3000/mcp

# SSE服务器（带OAuth）
claude mcp add --transport sse my-sse-server https://api.example.com/sse \
  --client-id my-client-id \
  --client-secret

# 指定作用域
claude mcp add --scope user my-server -- npx my-mcp-server
claude mcp add --scope project my-server -- npx my-mcp-server
```

**手动配置文件：**

```json
// .mcp.json
{
  "mcpServers": {
    "stdio-server": {
      "command": "npx",
      "args": ["-y", "my-mcp-server"],
      "env": {
        "API_KEY": "${API_KEY}"  // 支持环境变量展开
      }
    },
    "http-server": {
      "type": "http",
      "url": "https://api.example.com/mcp",
      "headers": {
        "X-Custom-Header": "value"
      },
      "oauth": {
        "clientId": "my-client-id",
        "callbackPort": 8080,
        "authServerMetadataUrl": "https://auth.example.com/.well-known/oauth-authorization-server"
      }
    },
    "sse-server": {
      "type": "sse",
      "url": "https://api.example.com/sse",
      "headersHelper": "helper-script.sh"  // 动态获取headers
    }
  }
}
```

### 7.4 进程内MCP服务器

**使用InProcessTransport：**

```typescript
// 来自 services/mcp/InProcessTransport.ts
import { createLinkedTransportPair } from './InProcessTransport.js'

// 创建进程内传输对
const [clientTransport, serverTransport] = createLinkedTransportPair()

// 连接服务器
const server = new Server(
  { name: 'in-process-server', version: '1.0.0' },
  { capabilities: { tools: {} } },
)
await server.connect(serverTransport)

// 连接客户端
const client = new Client(
  { name: 'in-process-client', version: '1.0.0' },
  { capabilities: {} },
)
await client.connect(clientTransport)

// 现在可以直接调用工具
const result = await client.request(
  {
    method: 'tools/call',
    params: {
      name: 'echo',
      arguments: { message: 'Hello' },
    },
  },
  CallToolResultSchema,
)
```

### 7.5 实际示例：Claude Code内置MCP服务器

**来自 `entrypoints/mcp.ts`：**

```typescript
// 来自 entrypoints/mcp.ts:35-195
export async function startMCPServer(
  cwd: string,
  debug: boolean,
  verbose: boolean,
): Promise<void> {
  // 1. 设置工作目录
  setCwd(cwd)
  
  // 2. 创建服务器实例
  const server = new Server(
    {
      name: 'claude/tengu',
      version: MACRO.VERSION,
    },
    {
      capabilities: {
        tools: {},  // 暴露所有内置工具
      },
    },
  )
  
  // 3. 注册工具列表处理器
  server.setRequestHandler(
    ListToolsRequestSchema,
    async (): Promise<ListToolsResult> => {
      const toolPermissionContext = getEmptyToolPermissionContext()
      const tools = getTools(toolPermissionContext)
      
      // 转换工具为MCP格式
      return {
        tools: await Promise.all(
          tools.map(async tool => {
            // 转换Schema为JSON Schema格式
            const inputSchema = zodToJsonSchema(tool.inputSchema)
            const outputSchema = tool.outputSchema 
              ? zodToJsonSchema(tool.outputSchema) 
              : undefined
            
            return {
              name: tool.name,
              description: await tool.prompt({
                getToolPermissionContext: async () => toolPermissionContext,
                tools,
                agents: [],
              }),
              inputSchema,
              outputSchema,
            }
          }),
        ),
      }
    },
  )
  
  // 4. 注册工具调用处理器
  server.setRequestHandler(
    CallToolRequestSchema,
    async ({ params: { name, arguments: args } }): Promise<CallToolResult> => {
      // 查找工具
      const tools = getTools(getEmptyToolPermissionContext())
      const tool = findToolByName(tools, name)
      
      if (!tool) {
        throw new Error(`Tool ${name} not found`)
      }
      
      // 构建调用上下文
      const toolUseContext: ToolUseContext = {
        abortController: createAbortController(),
        options: {
          commands: [review],
          tools,
          mainLoopModel: getMainLoopModel(),
          thinkingConfig: { type: 'disabled' },
          mcpClients: [],
          mcpResources: {},
          isNonInteractiveSession: true,
          debug,
          verbose,
          agentDefinitions: { activeAgents: [], allAgents: [] },
        },
        getAppState: () => getDefaultAppState(),
        messages: [],
        readFileState: readFileStateCache,
      }
      
      try {
        // 执行工具调用
        const result = await tool.call(
          args ?? {},
          toolUseContext,
          hasPermissionsToUseTool,
          createAssistantMessage({ content: [] }),
        )
        
        return {
          content: [
            {
              type: 'text',
              text: typeof result === 'string' 
                ? result 
                : jsonStringify(result.data),
            },
          ],
        }
      } catch (error) {
        // 错误处理
        const parts = error instanceof Error 
          ? getErrorParts(error) 
          : [String(error)]
        
        return {
          isError: true,
          content: [
            {
              type: 'text',
              text: parts.filter(Boolean).join('\n').trim() || 'Error',
            },
          ],
        }
      }
    },
  )
  
  // 5. 启动服务器（stdio传输）
  const transport = new StdioServerTransport()
  await server.connect(transport)
}
```

---

## 8. 高级特性

### 8.1 动态Headers支持

**headersHelper机制：**

```typescript
// 来自 services/mcp/headersHelper.ts
export async function getMcpServerHeaders(
  name: string,
  config: ScopedMcpServerConfig,
): Promise<Record<string, string>> {
  // 1. 合并静态headers
  const staticHeaders = config.headers ?? {}
  
  // 2. 执行headersHelper脚本（如果配置）
  if (config.headersHelper) {
    const dynamicHeaders = await executeHeadersHelper(config.headersHelper)
    return { ...staticHeaders, ...dynamicHeaders }
  }
  
  return staticHeaders
}

// headersHelper脚本示例 (shell script)
#!/bin/bash
# 输出JSON格式的headers
echo '{"Authorization": "Bearer $(cat ~/.token)"}'
```

### 8.2 环境变量展开

**展开配置中的环境变量：**

```typescript
// 来自 services/mcp/envExpansion.ts
export function expandEnvVarsInString(value: string): string {
  // 支持 ${VAR} 和 $VAR 格式
  return value.replace(/\$\{([^}]+)\}/g, (_, varName) => {
    return process.env[varName] || ''
  })
}

// 配置展开示例
const config = {
  env: {
    API_KEY: '${API_KEY}',           // 从环境变量展开
    URL: 'https://${API_HOST}/api',  // 部分展开
  },
}
```

### 8.3 企业策略控制

**允许/拒绝列表：**

```typescript
// 来自 services/mcp/config.ts:364-408
function isMcpServerDenied(
  serverName: string,
  config?: McpServerConfig,
): boolean {
  const settings = getMcpDenylistSettings()
  
  if (!settings.deniedMcpServers) {
    return false
  }
  
  // 检查名称拒绝
  for (const entry of settings.deniedMcpServers) {
    if (isMcpServerNameEntry(entry) && entry.serverName === serverName) {
      return true
    }
  }
  
  // 检查命令拒绝（stdio服务器）
  if (config) {
    const serverCommand = getServerCommandArray(config)
    if (serverCommand) {
      for (const entry of settings.deniedMcpServers) {
        if (
          isMcpServerCommandEntry(entry) &&
          commandArraysMatch(entry.serverCommand, serverCommand)
        ) {
          return true
        }
      }
    }
  }
  
  // 检查URL拒绝（远程服务器，支持通配符）
  const serverUrl = getServerUrl(config)
  if (serverUrl) {
    for (const entry of settings.deniedMcpServers) {
      if (
        isMcpServerUrlEntry(entry) &&
        urlMatchesPattern(serverUrl, entry.serverUrl)
      ) {
        return true
      }
    }
  }
  
  return false
}
```

**策略配置示例：**

```json
// settings.json (企业策略)
{
  "allowedMcpServers": [
    { "serverName": "approved-server" },
    { "serverCommand": ["npx", "-y", "approved-tool"] },
    { "serverUrl": "https://approved.example.com/*" }
  ],
  "deniedMcpServers": [
    { "serverName": "blocked-server" },
    { "serverUrl": "https://blocked.example.com/*" }
  ]
}
```

### 8.4 资源订阅机制

**资源变更通知：**

```typescript
// 来自 services/mcp/client.ts
// 注册资源列表变更通知
client.setNotificationHandler(
  ResourceListChangedNotificationSchema,
  async (notification) => {
    // 重新获取资源列表
    const newResources = await fetchResourcesForClient(client, serverName)
    
    // 更新AppState
    setAppState(prev => ({
      ...prev,
      mcp: {
        ...prev.mcp,
        resources: {
          ...prev.mcp.resources,
          [serverName]: newResources,
        },
      },
    }))
  },
)
```

### 8.5 工具列表变更通知

**动态工具更新：**

```typescript
// 注册工具列表变更通知
client.setNotificationHandler(
  ToolListChangedNotificationSchema,
  async (notification) => {
    // 重新获取工具列表
    const newTools = await fetchToolsForClient(client, serverName)
    
    // 更新AppState（替换旧工具）
    setAppState(prev => ({
      ...prev,
      mcp: {
        ...prev.mcp,
        tools: [
          ...excludeToolsByServer(prev.mcp.tools, serverName),
          ...newTools,
        ],
      },
    }))
  },
)
```

---

## 9. 调试与日志

### 9.1 MCP调试日志

**启用MCP调试：**

```bash
# 设置环境变量启用调试日志
export MCP_DEBUG=1

# 或者在配置中指定
claude --mcp-debug
```

**调试日志输出：**

```typescript
// 来自 services/mcp/client.ts
import { logMCPDebug, logMCPError } from '../../utils/log.js'

// 连接调试
logMCPDebug(serverName, `SSE transport initialized, awaiting connection`)
logMCPDebug(serverName, `Node version: ${process.version}, Platform: ${process.platform}`)

// 错误日志
logMCPError(serverName, `Connection failed: ${errorMessage(error)}`)
```

### 9.2 性能监控

**连接统计：**

```typescript
// 来自 services/mcp/client.ts
interface ServerStats {
  totalServers: number
  stdioCount: number
  sseCount: number
  httpCount: number
  sseIdeCount: number
  wsIdeCount: number
}

// 发送分析事件
logEvent('tengu_mcp_server_connected', {
  transportType: transportType as AnalyticsMetadata,
  connectionTime: Date.now() - connectStartTime,
  ...mcpBaseUrlAnalytics(serverRef),
})
```

---

## 10. 安全考虑

### 10.1 项目MCP服务器审批

**审批流程（`services/mcp/utils.ts`）：**

```typescript
// 来自 services/mcp/utils.ts:351-406
export function getProjectMcpServerStatus(
  serverName: string,
): 'approved' | 'rejected' | 'pending' {
  const settings = getSettings_DEPRECATED()
  const normalizedName = normalizeNameForMCP(serverName)
  
  // 检查拒绝列表
  if (settings?.disabledMcpjsonServers?.some(
    name => normalizeNameForMCP(name) === normalizedName
  )) {
    return 'rejected'
  }
  
  // 检查批准列表
  if (
    settings?.enabledMcpjsonServers?.some(
      name => normalizeNameForMCP(name) === normalizedName
    ) ||
    settings?.enableAllProjectMcpServers
  ) {
    return 'approved'
  }
  
  // 非交互模式自动批准（需projectSettings启用）
  if (
    hasSkipDangerousModePermissionPrompt() &&
    isSettingSourceEnabled('projectSettings')
  ) {
    return 'approved'
  }
  
  return 'pending'  // 需要用户交互批准
}
```

### 10.2 OAuth安全措施

**URL参数敏感信息过滤：**

```typescript
// 来自 services/mcp/auth.ts:99-124
const SENSITIVE_OAUTH_PARAMS = [
  'state',
  'nonce',
  'code_challenge',
  'code_verifier',
  'code',
]

function redactSensitiveUrlParams(url: string): string {
  try {
    const parsedUrl = new URL(url)
    for (const param of SENSITIVE_OAUTH_PARAMS) {
      if (parsedUrl.searchParams.has(param)) {
        parsedUrl.searchParams.set(param, '[REDACTED]')
      }
    }
    return parsedUrl.toString()
  } catch {
    return url
  }
}
```

### 10.3 HTTPS强制要求

**强制HTTPS验证：**

```typescript
// 来自 services/mcp/auth.ts:263-268
if (configuredMetadataUrl) {
  if (!configuredMetadataUrl.startsWith('https://')) {
    throw new Error(
      `authServerMetadataUrl must use https:// (got: ${configuredMetadataUrl})`,
    )
  }
  // ...获取元数据
}
```

---

## 11. 总结

### 11.1 MCP服务器开发要点

**核心要点总结：**

1. **协议理解:** MCP基于JSON-RPC 2.0，使用SDK简化开发
2. **传输选择:** stdio适合本地工具，HTTP/SSE适合远程服务
3. **工具定义:** 清晰的inputSchema/outputSchema，遵循命名规范
4. **认证集成:** OAuth 2.0标准流程，支持动态发现
5. **错误处理:** 分类错误类型，提供清晰错误信息
6. **资源管理:** 合理清理资源，避免内存泄漏
7. **配置管理:** 理解作用域层次，正确配置路径

### 11.2 最佳实践清单

**开发MCP服务器时的检查清单：**

- [ ] 使用最新的MCP SDK版本
- [ ] 提供清晰的工具描述和Schema
- [ ] 实现完整的错误处理
- [ ] 支持工具列表变更通知
- [ ] 正确处理超时和中断
- [ ] 清理所有资源（连接、缓存）
- [ ] 使用HTTPS进行认证
- [ ] 测试各种传输类型
- [ ] 记录足够的调试日志
- [ ] 遵循命名规范（mcp__server__tool）

### 11.3 相关文件索引

**核心文件路径：**

```
services/mcp/
├── types.ts              # 配置类型定义
├── config.ts             # 配置加载管理
├── client.ts             # 客户端连接逻辑
├── auth.ts               # OAuth认证实现
├── utils.ts              # 工具函数集合
├── normalization.ts      # 名称规范化
├── mcpStringUtils.ts     # 字符串解析工具
├── InProcessTransport.ts # 进程内传输
├── useManageMCPConnections.ts # React连接管理hook
├── xaa.ts                # Cross-App Access
└── xaaIdpLogin.ts        # IdP登录集成

tools/MCPTool/
└── MCPTool.ts            # MCP工具基础实现

commands/mcp/
├── addCommand.ts         # mcp add命令
├── index.ts              # mcp命令入口
└── xaaIdpCommand.ts      # XAA IdP配置命令

entrypoints/
└── mcp.ts                # MCP服务器入口点
```

---

**文档版本:** 1.0  
**最后更新:** 2026-04-02  
**适用版本:** Claude Code CLI v1.x