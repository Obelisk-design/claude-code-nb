# MCP集成机制深度解析

**分析日期:** 2026-04-02

## 1. MCP概述

### 1.1 Model Context Protocol介绍

Model Context Protocol (MCP) 是Anthropic设计的一种开放协议，用于AI应用与外部工具、数据源之间的标准化通信。它解决了AI模型与外部系统交互的核心问题：

- **工具调用标准化**: 定义统一的工具发现、调用和响应格式
- **资源访问协议**: 提供结构化的资源读取和订阅机制
- **提示模板管理**: 支持动态提示模板的发现和执行
- **认证集成**: 内置OAuth 2.0认证流程支持

MCP协议核心概念：
```
┌─────────────────┐     MCP Protocol     ┌─────────────────┐
│   Claude Code   │ ◄──────────────────► │   MCP Server    │
│    (Client)     │                      │   (Provider)    │
└─────────────────┘                      └─────────────────┘
        │                                       │
        │  • tools/list → 工具发现               │
        │  • tools/call → 工具调用               │
        │  • resources/list → 资源发现           │
        │  • prompts/list → 提示发现             │
        └                                       │
```

### 1.2 MCP在Claude Code中的作用

Claude Code通过MCP实现以下能力扩展：

| 功能 | 实现文件 | 说明 |
|------|----------|------|
| 工具扩展 | `services/mcp/client.ts` | 动态发现并注册MCP服务器提供的工具 |
| 资源访问 | `services/mcp/client.ts` | 读取MCP服务器暴露的文件、数据库等资源 |
| 提示模板 | `services/mcp/client.ts` | 使用MCP服务器提供的预定义提示模板 |
| 认证管理 | `services/mcp/auth.ts` | OAuth 2.0认证流程和令牌管理 |
| 配置管理 | `services/mcp/config.ts` | 多来源配置合并和作用域管理 |

---

## 2. services/mcp/client.ts 逐行分析

### 2.1 第1-50行：导入和核心依赖

```typescript
// 第1-6行：MCP SDK核心导入
import { Client } from '@modelcontextprotocol/sdk/client/index.js'
import { SSEClientTransport } from '@modelcontextprotocol/sdk/client/sse.js'
import { StdioClientTransport } from '@modelcontextprotocol/sdk/client/stdio.js'
import { StreamableHTTPClientTransport } from '@modelcontextprotocol/sdk/client/streamableHttp.js'
```

**关键依赖解析：**
- `Client`: MCP客户端主类，管理连接生命周期
- `SSEClientTransport`: Server-Sent Events传输层，用于远程服务器
- `StdioClientTransport`: 标准输入输出传输层，用于本地进程
- `StreamableHTTPClientTransport`: HTTP流式传输，支持双向通信

```typescript
// 第39-42行：工具函数导入
import mapValues from 'lodash-es/mapValues.js'
import memoize from 'lodash-es/memoize.js'
import zipObject from 'lodash-es/zipObject.js'
import pMap from 'p-map'
```

**设计意图：**
- `memoize`: 缓存连接结果，避免重复连接
- `pMap`: 并行批量连接MCP服务器，控制并发数

```typescript
// 第54-56行：MCP工具桥接
import { MCPTool } from '../../tools/MCPTool/MCPTool.js'
import { createMcpAuthTool } from '../../tools/McpAuthTool/McpAuthTool.js'
import { ReadMcpResourceTool } from '../../tools/ReadMcpResourceTool/ReadMcpResourceTool.js'
```

这些工具将MCP能力映射到Claude Code的Tool系统。

### 2.2 第50-150行：错误类型和认证处理

```typescript
// 第152-159行：McpAuthError定义
export class McpAuthError extends Error {
  serverName: string
  constructor(serverName: string, message: string) {
    super(message)
    this.name = 'McpAuthError'
    this.serverName = serverName
  }
}
```

**设计意图：**
- 自定义错误类携带服务器名称，便于错误定位
- 在工具执行层捕获后更新客户端状态为 `needs-auth`

```typescript
// 第165-170行：会话过期错误
class McpSessionExpiredError extends Error {
  constructor(serverName: string) {
    super(`MCP server "${serverName}" session expired`)
    this.name = 'McpSessionExpiredError'
  }
}
```

HTTP传输的会话ID过期时抛出，触发重新连接。

```typescript
// 第193-206行：会话过期检测
export function isMcpSessionExpiredError(error: Error): boolean {
  const httpStatus = 'code' in error ? error.code : undefined
  if (httpStatus !== 404) return false
  // 检查JSON-RPC错误码 -32001
  return error.message.includes('"code":-32001') ||
         error.message.includes('"code": -32001')
}
```

**检测逻辑：**
1. HTTP状态码必须是404
2. 错误消息包含MCP特定的JSON-RPC错误码-32001
3. 双重验证避免误判普通404错误

```typescript
// 第211-218行：常量定义
const DEFAULT_MCP_TOOL_TIMEOUT_MS = 100_000_000  // ~27.8小时
const MAX_MCP_DESCRIPTION_LENGTH = 2048          // 描述截断长度
```

**设计考量：**
- 工具超时设为极大值，让MCP服务器自行控制执行时间
- 描述截断防止OpenAPI生成的超长文档消耗上下文

### 2.3 第150-300行：认证缓存机制

```typescript
// 第257-263行：认证缓存配置
const MCP_AUTH_CACHE_TTL_MS = 15 * 60 * 1000  // 15分钟
type McpAuthCacheData = Record<string, { timestamp: number }>

function getMcpAuthCachePath(): string {
  return join(getClaudeConfigHomeDir(), 'mcp-needs-auth-cache.json')
}
```

**缓存目的：**
- 避免频繁弹出认证提示
- 记录需要认证的服务器，15分钟内不再重复尝试

```typescript
// 第269-278行：内存缓存优化
let authCachePromise: Promise<McpAuthCacheData> | null = null

function getMcpAuthCache(): Promise<McpAuthCacheData> {
  if (!authCachePromise) {
    authCachePromise = readFile(getMcpAuthCachePath(), 'utf-8')
      .then(data => jsonParse(data) as McpAuthCacheData)
      .catch(() => ({}))
  }
  return authCachePromise
}
```

**设计模式：**
- 单例Promise模式，避免并发读取同一文件
- 写入时清空缓存 (`authCachePromise = null`)，确保下次读取最新数据

```typescript
// 第289-309行：序列化写入防止竞态
let writeChain = Promise.resolve()

function setMcpAuthCacheEntry(serverId: string): void {
  writeChain = writeChain
    .then(async () => {
      const cache = await getMcpAuthCache()
      cache[serverId] = { timestamp: Date.now() }
      await writeFile(...)
      authCachePromise = null  // 清空读缓存
    })
}
```

**竞态处理：**
- Promise链序列化所有写入操作
- 读后写模式确保数据一致性

### 2.4 第300-400行：远程认证失败处理

```typescript
// 第340-361行：统一认证失败处理
function handleRemoteAuthFailure(
  name: string,
  serverRef: ScopedMcpServerConfig,
  transportType: 'sse' | 'http' | 'claudeai-proxy',
): MCPServerConnection {
  logEvent('tengu_mcp_server_needs_auth', {...})
  setMcpAuthCacheEntry(name)
  return { name, type: 'needs-auth', config: serverRef }
}
```

**处理流程：**
1. 记录分析事件
2. 缓存认证需求状态
3. 返回 `needs-auth` 类型连接对象

```typescript
// 第372-422行：claude.ai代理认证
export function createClaudeAiProxyFetch(innerFetch: FetchLike): FetchLike {
  return async (url, init) => {
    const doRequest = async () => {
      await checkAndRefreshOAuthTokenIfNeeded()
      const currentTokens = getClaudeAIOAuthTokens()
      headers.set('Authorization', `Bearer ${currentTokens.accessToken}`)
      return { response, sentToken: currentTokens.accessToken }
    }
    // 401时重试一次
    if (response.status !== 401) return response
    const tokenChanged = await handleOAuth401Error(sentToken)
    if (tokenChanged) return (await doRequest()).response
  }
}
```

**认证重试策略：**
1. 首次请求携带当前令牌
2. 401响应时尝试刷新令牌
3. 仅在令牌实际变更时重试

### 2.5 传输层实现分析

#### Stdio传输（本地进程）

```typescript
// 第944-959行：Stdio传输初始化
transport = new StdioClientTransport({
  command: finalCommand,
  args: finalArgs,
  env: { ...subprocessEnv(), ...serverRef.env },
  stderr: 'pipe',  // 捕获错误输出
})
```

**特点：**
- 通过stdin/stdout通信
- 支持自定义环境变量
- stderr独立管道，不干扰协议通信

#### SSE传输（Server-Sent Events）

```typescript
// 第619-676行：SSE传输初始化
const transportOptions: SSEClientTransportOptions = {
  authProvider,
  fetch: wrapFetchWithTimeout(
    wrapFetchWithStepUpDetection(createFetchWithInit(), authProvider)
  ),
  eventSourceInit: {
    fetch: async (url, init) => {
      // EventSource连接不应用超时（长连接）
      const authHeaders = {}
      const tokens = await authProvider.tokens()
      if (tokens) authHeaders.Authorization = `Bearer ${tokens.access_token}`
      return fetch(url, { ...init, headers: { ...authHeaders, Accept: 'text/event-stream' } })
    }
  }
}
transport = new SSEClientTransport(new URL(serverRef.url), transportOptions)
```

**关键设计：**
- GET请求（EventSource）不设超时，保持长连接
- POST请求应用60秒超时
- 认证Provider自动注入Bearer令牌

#### HTTP传输（Streamable HTTP）

```typescript
// 第784-865行：HTTP传输初始化
const transportOptions: StreamableHTTPClientTransportOptions = {
  authProvider,
  fetch: wrapFetchWithTimeout(
    wrapFetchWithStepUpDetection(createFetchWithInit(), authProvider)
  ),
  requestInit: {
    headers: {
      'User-Agent': getMCPUserAgent(),
      ...(sessionIngressToken && { Authorization: `Bearer ${sessionIngressToken}` }),
      ...combinedHeaders,
    }
  }
}
transport = new StreamableHTTPClientTransport(new URL(serverRef.url), transportOptions)
```

**MCP Streamable HTTP规范：**
- POST请求Accept头必须包含 `application/json, text/event-stream`
- 支持会话ID管理，过期时需重新连接

#### WebSocket传输

```typescript
// 第735-783行：WebSocket传输初始化
let wsClient: WsClientLike
if (typeof Bun !== 'undefined') {
  wsClient = new globalThis.WebSocket(serverRef.url, {
    protocols: ['mcp'],
    headers: wsHeaders,
    proxy: getWebSocketProxyUrl(serverRef.url),
    tls: tlsOptions,
  })
} else {
  wsClient = await createNodeWsClient(serverRef.url, {
    headers: wsHeaders,
    agent: getWebSocketProxyAgent(serverRef.url),
    ...tlsOptions,
  })
}
transport = new WebSocketTransport(wsClient)
```

**双运行时支持：**
- Bun环境使用内置WebSocket
- Node.js环境使用ws包

---

## 3. MCP相关文件分析

### 3.1 services/mcp/types.ts 类型定义

#### 配置作用域

```typescript
// 第10-21行
export const ConfigScopeSchema = lazySchema(() =>
  z.enum(['local', 'user', 'project', 'dynamic', 'enterprise', 'claudeai', 'managed'])
)
export type ConfigScope = z.infer<ReturnType<typeof ConfigScopeSchema>>
```

| 作用域 | 来源 | 用途 |
|--------|------|------|
| `local` | 本地私有配置 | 仅当前项目可见 |
| `user` | 用户全局配置 | 所有项目共享 |
| `project` | .mcp.json文件 | 项目级别共享 |
| `dynamic` | 动态配置 | 插件或命令行提供 |
| `enterprise` | 企业管理配置 | 组织统一管理 |
| `claudeai` | claude.ai代理 | 云端服务 |
| `managed` | 受管配置 | 企业策略控制 |

#### 传输类型

```typescript
// 第23-26行
export const TransportSchema = lazySchema(() =>
  z.enum(['stdio', 'sse', 'sse-ide', 'http', 'ws', 'sdk'])
)
```

#### 服务器配置Schema

```typescript
// 第28-135行：各传输类型配置Schema
McpStdioServerConfigSchema    // command, args, env
McpSSEServerConfigSchema      // url, headers, oauth
McpHTTPServerConfigSchema     // url, headers, oauth
McpWebSocketServerConfigSchema// url, headers
McpSdkServerConfigSchema      // name (SDK内置)
McpClaudeAIProxyServerConfigSchema // url, id (代理)
```

#### 连接状态类型

```typescript
// 第180-227行：连接状态联合类型
export type ConnectedMCPServer = {
  client: Client
  name: string
  type: 'connected'
  capabilities: ServerCapabilities
  serverInfo?: { name: string; version: string }
  instructions?: string
  config: ScopedMcpServerConfig
  cleanup: () => Promise<void>
}

export type FailedMCPServer = { name: string; type: 'failed'; config: ...; error?: string }
export type NeedsAuthMCPServer = { name: string; type: 'needs-auth'; config: ... }
export type PendingMCPServer = { name: string; type: 'pending'; config: ... }
export type DisabledMCPServer = { name: string; type: 'disabled'; config: ... }

export type MCPServerConnection = 
  | ConnectedMCPServer | FailedMCPServer | NeedsAuthMCPServer 
  | PendingMCPServer | DisabledMCPServer
```

### 3.2 services/mcp/auth.ts 认证机制

#### OAuth元数据发现

```typescript
// 第256-310行
async function fetchAuthServerMetadata(
  serverName: string,
  serverUrl: string,
  configuredMetadataUrl: string | undefined,
): Promise<AuthorizationServerMetadata> {
  if (configuredMetadataUrl) {
    // 用户配置的元数据URL（必须HTTPS）
    const response = await authFetch(configuredMetadataUrl, { headers: { Accept: 'application/json' } })
    return OAuthMetadataSchema.parse(await response.json())
  }
  // RFC 9728 → RFC 8414 自动发现
  const { authorizationServerMetadata } = await discoverOAuthServerInfo(serverUrl, ...)
}
```

**发现顺序：**
1. 用户配置的 `authServerMetadataUrl`
2. RFC 9728: `/.well-known/oauth-protected-resource` → 获取授权服务器URL
3. RFC 8414: `/.well-known/oauth-authorization-server/{path}`

#### 令牌撤销

```typescript
// 第467-576行
export async function revokeServerTokens(
  serverName: string,
  serverConfig: McpSSEServerConfig | McpHTTPServerConfig,
): Promise<void> {
  // 先撤销refresh_token（防止生成新access_token）
  await revokeToken({ token: tokenData.refreshToken, tokenTypeHint: 'refresh_token' })
  // 再撤销access_token
  await revokeToken({ token: tokenData.accessToken, tokenTypeHint: 'access_token' })
  // 清除本地存储
  clearServerTokensFromLocalStorage(serverName, serverConfig)
}
```

**RFC 7009合规：**
- 先撤销长效令牌（refresh_token）
- 支持多种认证方法（client_secret_basic, client_secret_post）
- 401时回退到Bearer认证

#### 错误响应规范化

```typescript
// 第157-190行
export async function normalizeOAuthErrorBody(response: Response): Promise<Response> {
  // 处理HTTP 200但包含错误的响应（如Slack）
  if (!response.ok) return response
  const parsed = jsonParse(await response.text())
  if (OAuthTokensSchema.safeParse(parsed).success) return response  // 成功响应
  if (OAuthErrorResponseSchema.safeParse(parsed).success) {
    // 非标准错误码标准化为invalid_grant
    const normalized = NONSTANDARD_INVALID_GRANT_ALIASES.has(parsed.error)
      ? { error: 'invalid_grant', ... }
      : parsed
    return new Response(jsonStringify(normalized), { status: 400 })
  }
}
```

### 3.3 services/mcp/officialRegistry.ts 官方注册

```typescript
// 第33-60行：预取官方MCP服务器列表
export async function prefetchOfficialMcpUrls(): Promise<void> {
  const response = await axios.get(
    'https://api.anthropic.com/mcp-registry/v0/servers?version=latest&visibility=commercial'
  )
  const urls = new Set<string>()
  for (const entry of response.data.servers) {
    for (const remote of entry.server.remotes ?? []) {
      urls.add(normalizeUrl(remote.url))  // 移除查询参数和尾部斜杠
    }
  }
  officialUrls = urls
}

// 第66-68行：URL验证
export function isOfficialMcpUrl(normalizedUrl: string): boolean {
  return officialUrls?.has(normalizedUrl) ?? false  // fail-closed
}
```

**用途：**
- 识别官方认证的MCP服务器
- 安全策略区分官方和第三方服务器

---

## 4. MCP工具桥接

### 4.1 MCPTool实现

文件位置: `tools/MCPTool/MCPTool.ts`

```typescript
// 第27-77行：MCPTool基础定义
export const MCPTool = buildTool({
  isMcp: true,
  name: 'mcp',  // 被client.ts动态覆盖
  maxResultSizeChars: 100_000,
  async description() { return DESCRIPTION },  // 被覆盖
  async prompt() { return PROMPT },            // 被覆盖
  async call() { return { data: '' } },        // 被覆盖
  renderToolUseMessage,
  renderToolUseProgressMessage,
  renderToolResultMessage,
})
```

**设计模式：**
- 定义基础骨架和渲染方法
- 在 `client.ts` 的 `fetchToolsForClient` 中动态覆盖

### 4.2 工具名称映射

文件位置: `services/mcp/mcpStringUtils.ts`

```typescript
// 第50-52行：构建完全限定名
export function buildMcpToolName(serverName: string, toolName: string): string {
  return `mcp__${normalizeNameForMCP(serverName)}__${normalizeNameForMCP(toolName)}`
}

// 第19-32行：解析工具名
export function mcpInfoFromString(toolString: string): { serverName: string; toolName: string } | null {
  const parts = toolString.split('__')
  const [mcpPart, serverName, ...toolNameParts] = parts
  if (mcpPart !== 'mcp' || !serverName) return null
  return { serverName, toolName: toolNameParts.join('__') }
}
```

**命名规则：**
- 格式: `mcp__<serverName>__<toolName>`
- 非法字符替换为下划线
- 限制: 服务器名不能包含 `__`（解析限制）

### 4.3 工具发现流程

文件位置: `services/mcp/client.ts` 第1743-1999行

```typescript
export const fetchToolsForClient = memoizeWithLRU(
  async (client: MCPServerConnection): Promise<Tool[]> => {
    if (client.type !== 'connected') return []
    if (!client.capabilities?.tools) return []
    
    // 调用MCP协议的tools/list
    const result = await client.client.request(
      { method: 'tools/list' },
      ListToolsResultSchema
    )
    
    // 转换为Claude Code Tool格式
    return toolsToProcess.map((tool): Tool => {
      const fullyQualifiedName = buildMcpToolName(client.name, tool.name)
      return {
        ...MCPTool,  // 继承基础定义
        name: skipPrefix ? tool.name : fullyQualifiedName,
        mcpInfo: { serverName: client.name, toolName: tool.name },
        async call(args, context, ...) {
          const connectedClient = await ensureConnectedClient(client)
          const mcpResult = await callMCPToolWithUrlElicitationRetry(...)
          return { data: mcpResult.content }
        },
        userFacingName() {
          return `${client.name} - ${tool.annotations?.title || tool.name} (MCP)`
        }
      }
    })
  },
  (client) => client.name,
  MCP_FETCH_CACHE_SIZE  // 缓存20个服务器
)
```

---

## 5. 从零实现指南

### 5.1 MCP客户端设计架构

```
┌─────────────────────────────────────────────────────────────┐
│                    MCP Client Architecture                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐      │
│  │   Config    │───►│  Transport  │───►│   Client    │      │
│  │   Layer     │    │   Layer     │    │   Layer     │      │
│  └─────┬───────┘    └─────┬───────┘    └─────┬───────┘      │
│        │                  │                  │              │
│        ▼                  ▼                  ▼              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐      │
│  │   Auth      │    │   Protocol  │    │   Tool      │      │
│  │   Provider  │    │   Handlers  │    │   Bridge    │      │
│  └─────────────┘    └─────────────┘    └─────────────┘      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 传输层选择策略

| 场景 | 推荐传输 | 原因 |
|------|----------|------|
| 本地CLI工具 | Stdio | 进程间通信高效 |
| 远程HTTP服务 | HTTP (Streamable) | 支持双向流式通信 |
| 实时推送需求 | SSE | 服务器主动推送 |
| 双向实时通信 | WebSocket | 全双工通信 |
| 内置功能 | SDK | 无需外部进程 |

**实现要点：**

```typescript
// 传输层选择逻辑
function selectTransport(config: McpServerConfig): Transport {
  switch (config.type) {
    case 'stdio':
      return new StdioClientTransport({
        command: config.command,
        args: config.args,
        env: mergeEnv(config.env),
      })
    case 'sse':
      return new SSEClientTransport(new URL(config.url), {
        authProvider: createAuthProvider(config),
        fetch: wrapWithTimeout(fetch),  // POST超时，GET不限
      })
    case 'http':
      return new StreamableHTTPClientTransport(new URL(config.url), {
        authProvider: createAuthProvider(config),
        fetch: wrapWithTimeout(fetch),
      })
    case 'ws':
      return new WebSocketTransport(createWebSocket(config.url))
  }
}
```

### 5.3 工具/资源发现实现

```typescript
// 工具发现核心流程
async function discoverTools(client: Client): Promise<Tool[]> {
  // 1. 检查服务器能力
  const capabilities = client.getServerCapabilities()
  if (!capabilities?.tools) return []
  
  // 2. 调用tools/list
  const result = await client.request(
    { method: 'tools/list' },
    ListToolsResultSchema
  )
  
  // 3. 转换并注册
  return result.tools.map(tool => ({
    name: `mcp__${serverName}__${tool.name}`,
    description: tool.description,
    inputSchema: tool.inputSchema,
    call: async (args) => {
      return client.callTool({ name: tool.name, arguments: args })
    }
  }))
}

// 资源发现核心流程
async function discoverResources(client: Client): Promise<Resource[]> {
  const capabilities = client.getServerCapabilities()
  if (!capabilities?.resources) return []
  
  const result = await client.request(
    { method: 'resources/list' },
    ListResourcesResultSchema
  )
  
  return result.resources.map(r => ({ ...r, server: serverName }))
}
```

### 5.4 认证集成实现

```typescript
// OAuth认证Provider
class ClaudeAuthProvider implements OAuthClientProvider {
  async clientMetadata(): Promise<OAuthClientMetadata> {
    return {
      client_name: 'Claude Code',
      redirect_uris: [buildRedirectUri(await findAvailablePort())],
      grant_types: ['authorization_code', 'refresh_token'],
    }
  }
  
  async tokens(): Promise<OAuthTokens | undefined> {
    const stored = readFromSecureStorage(serverKey)
    if (!stored?.accessToken) return undefined
    return { access_token: stored.accessToken, refresh_token: stored.refreshToken }
  }
  
  async saveTokens(tokens: OAuthTokens): Promise<void> {
    writeToSecureStorage(serverKey, {
      accessToken: tokens.access_token,
      refreshToken: tokens.refresh_token,
      expiresAt: Date.now() + (tokens.expires_in ?? 3600) * 1000,
    })
  }
}
```

### 5.5 连接生命周期管理

```typescript
// 连接建立
async function connectToServer(name: string, config: McpServerConfig) {
  const transport = selectTransport(config)
  const client = new Client({ name: 'my-app', version: '1.0' })
  
  // 设置超时
  const timeoutPromise = setTimeout(() => {
    transport.close()
    throw new Error('Connection timeout')
  }, getConnectionTimeoutMs())
  
  try {
    await client.connect(transport)
    clearTimeout(timeoutPromise)
    
    // 注册事件处理器
    client.onerror = (error) => handleConnectionError(error)
    client.onclose = () => clearConnectionCache(name)
    
    return { client, name, type: 'connected', capabilities: client.getServerCapabilities() }
  } catch (error) {
    clearTimeout(timeoutPromise)
    return { name, type: 'failed', error: error.message }
  }
}

// 清理与重连
async function cleanup(client: ConnectedMCPServer) {
  // Stdio进程需要优雅终止
  if (client.config.type === 'stdio') {
    process.kill(childPid, 'SIGINT')
    await sleep(100)
    process.kill(childPid, 'SIGTERM')
  }
  await client.client.close()
}
```

---

## 6. 关键设计模式总结

### 6.1 缓存策略

| 缓存类型 | 位置 | TTL | 清除时机 |
|----------|------|-----|----------|
| 连接缓存 | `connectToServer.memoize` | 无限 | 连接关闭 |
| 工具缓存 | `fetchToolsForClient.LRU` | 无限 | 服务器关闭/重载 |
| 认证缓存 | `mcp-needs-auth-cache.json` | 15分钟 | 认证成功 |

### 6.2 错误处理层级

```
Tool Execution Error
    │
    ├── McpAuthError → 状态变为 needs-auth
    ├── McpSessionExpiredError → 清缓存重连
    ├── McpError (SDK) → TelemetrySafeError包装
    └── Network Error → 指数退避重试
```

### 6.3 并发控制

```typescript
// 批量连接并发限制
const batchSize = getMcpServerConnectionBatchSize()  // 默认3
await pMap(servers, connectToServer, { concurrency: batchSize })

// 认证写入序列化
let writeChain = Promise.resolve()
writeChain = writeChain.then(() => writeCache())
```

---

## 7. 文件清单与职责

| 文件路径 | 核心职责 |
|----------|----------|
| `services/mcp/client.ts` | 连接管理、工具发现、资源管理 |
| `services/mcp/types.ts` | 类型定义、Schema验证 |
| `services/mcp/auth.ts` | OAuth认证、令牌管理、撤销 |
| `services/mcp/config.ts` | 配置加载、作用域合并 |
| `services/mcp/utils.ts` | 过滤工具、哈希配置、URL处理 |
| `services/mcp/normalization.ts` | 名称规范化 |
| `services/mcp/mcpStringUtils.ts` | 工具名解析/构建 |
| `services/mcp/officialRegistry.ts` | 官方服务器注册 |
| `services/mcp/useManageMCPConnections.ts` | React连接管理Hook |
| `tools/MCPTool/MCPTool.ts` | MCP工具桥接基类 |

---

*MCP集成机制分析完成*