# Bridge远程控制模式详解

**文档版本**: 2026-04-02  
**源码版本**: Claude Code CLI

---

## 目录

1. [Bridge模式概述](#1-bridge模式概述)
2. [bridge目录结构](#2-bridge目录结构)
3. [远程会话架构](#3-远程会话架构)
4. [消息传递机制](#4-消息传递机制)
5. [安全机制](#5-安全机制)
6. [用户使用指南](#6-用户使用指南)
7. [从零实现Bridge](#7-从零实现bridge)

---

## 1. Bridge模式概述

### 1.1 什么是Remote Control

Bridge远程控制模式（Remote Control）是Claude Code CLI的核心功能，允许用户通过 **claude.ai网站** 或 **移动应用** 远程控制本地终端中运行的Claude Code会话。

**核心能力**：
- 从手机/网页发送消息到本地终端
- 实时查看终端输出和工具执行
- 远程批准/拒绝权限请求
- 切换模型、中断执行
- 查看会话历史和状态

### 1.2 架构图

```
┌─────────────────┐                    ┌─────────────────┐
│  claude.ai      │                    │  Claude Code    │
│  (Web/Mobile)   │                    │  CLI (Local)    │
└─────────────────┘                    └─────────────────┘
        │                                      │
        │         ┌─────────────────┐          │
        │         │   CCR Server    │          │
        └────────►│ (Session-Ingress)◄─────────┘
                  │   /worker/*     │
                  └─────────────────┘
                          │
                  ┌───────┴───────┐
                  │  Bridge Mode  │
                  │  Transport    │
                  └───────────────┘
```

### 1.3 两种实现路径

Bridge模式有两种实现路径，通过GrowthBook开关 `tengu_bridge_repl_v2` 控制：

| 特性 | v1 (env-based) | v2 (env-less) |
|------|----------------|---------------|
| **API层级** | Environments API + Sessions API | 直接Sessions API |
| **生命周期** | register/poll/ack/heartbeat/deregister | 无环境生命周期 |
| **会话创建** | POST /v1/sessions | POST /v1/code/sessions |
| **认证获取** | poll返回work_secret | POST /bridge返回worker_jwt |
| **传输协议** | HybridTransport (WebSocket) | SSETransport + CCRClient |
| **适用场景** | daemon/print模式 | REPL模式 |

**v2优势**：
- 减少API调用层级（无poll循环）
- 更简单的认证流程
- 更好的错误恢复机制

---

## 2. bridge目录结构

### 2.1 核心文件概览

```
bridge/
├── types.ts                    # 类型定义（BridgeConfig, SessionHandle等）
├── bridgeMain.ts               # standalone bridge主入口
├── bridgeConfig.ts             # 配置读取（OAuth token, baseUrl）
├── bridgeEnabled.ts            # 功能开关检查
├── bridgeMessaging.ts          # 消息处理（入站/出站）
├── bridgeUI.ts                 # 终端UI显示
├── bridgeApi.ts                # API客户端封装
├── bridgeDebug.ts              # 调试工具
├── bridgePointer.ts            # crash-recovery指针
├── bridgeStatusUtil.ts         # 状态工具
├── bridgePermissionCallbacks.ts # 权限回调
│
├── remoteBridgeCore.ts         # v2 env-less核心实现
├── replBridge.ts               # v1 env-based核心实现
├── replBridgeTransport.ts      # 传输层抽象
├── replBridgeHandle.ts         # handle类型定义
├── initReplBridge.ts           # REPL初始化入口
│
├── codeSessionApi.ts           # v2会话API
├── createSession.ts            # v1会话创建
├── sessionRunner.ts            # 会话运行器
│
├── jwtUtils.ts                 # JWT刷新调度器
├── trustedDevice.ts            # 信任设备令牌
├── workSecret.ts               # work_secret解码
├── sessionIdCompat.ts          # session_*与cse_*转换
│
├── flushGate.ts                # 刷新门控
├── inboundMessages.ts          # 入站消息处理
├── inboundAttachments.ts       # 入站附件处理
├── capacityWake.ts             # 容量唤醒信号
│
├── envLessBridgeConfig.ts      # v2配置
├── pollConfig.ts               # poll间隔配置
├── pollConfigDefaults.ts       # 默认poll配置
│
├── debugUtils.ts               # 调试工具函数
└── bridgePermissionCallbacks.ts # 权限回调
```

### 2.2 文件职责详解

#### `types.ts` - 类型定义

定义了Bridge模式所需的所有核心类型：

```typescript
// 会话超时配置
export const DEFAULT_SESSION_TIMEOUT_MS = 24 * 60 * 60 * 1000 // 24小时

// Bridge配置
export type BridgeConfig = {
  dir: string                  // 工作目录
  machineName: string          // 机器名称
  branch: string               // Git分支
  gitRepoUrl: string | null    // Git仓库URL
  maxSessions: number          // 最大会话数
  spawnMode: SpawnMode         // 启动模式
  verbose: boolean             // 详细日志
  sandbox: boolean             // 沙箱模式
  bridgeId: string             // Bridge实例UUID
  workerType: string           // 工作类型标识
  environmentId: string        // 环境UUID
  reuseEnvironmentId?: string  // 重用的环境ID
  apiBaseUrl: string           // API基础URL
  sessionIngressUrl: string    // Session入口URL
  sessionTimeoutMs?: number    // 会话超时
}

// 会话句柄
export type SessionHandle = {
  sessionId: string            // 会话ID
  done: Promise<SessionDoneStatus> // 完成状态Promise
  kill(): void                 // 正常终止
  forceKill(): void            // 强制终止
  activities: SessionActivity[] // 活动记录
  currentActivity: SessionActivity | null // 当前活动
  accessToken: string          // 访问令牌
  lastStderr: string[]         // stderr缓冲
  writeStdin(data: string): void // 写入stdin
  updateAccessToken(token: string): void // 更新令牌
}
```

#### `bridgeEnabled.ts` - 功能开关

检查Remote Control是否可用：

```typescript
export function isBridgeEnabled(): boolean {
  return feature('BRIDGE_MODE')
    ? isClaudeAISubscriber() &&
        getFeatureValue_CACHED_MAY_BE_STALE('tengu_ccr_bridge', false)
    : false
}

export async function isBridgeEnabledBlocking(): Promise<boolean> {
  return feature('BRIDGE_MODE')
    ? isClaudeAISubscriber() &&
        (await checkGate_CACHED_OR_BLOCKING('tengu_ccr_bridge'))
    : false
}

// v2路径开关
export function isEnvLessBridgeEnabled(): boolean {
  return feature('BRIDGE_MODE')
    ? getFeatureValue_CACHED_MAY_BE_STALE('tengu_bridge_repl_v2', false)
    : false
}
```

**启用条件**：
1. 构建时包含 `BRIDGE_MODE` 特性标志
2. 用户是claude.ai订阅者
3. GrowthBook开关 `tengu_ccr_bridge` 为true

---

## 3. 远程会话架构

### 3.1 会话生命周期

#### v1 (env-based) 流程

```
1. registerBridgeEnvironment → environment_id, environment_secret
2. createSession → session_id
3. pollForWork → WorkResponse (work_secret)
4. decodeWorkSecret → session_ingress_token, api_base_url
5. connect WebSocket (HybridTransport)
6. [运行会话...]
7. acknowledgeWork (可选)
8. heartbeatWork (定期)
9. archiveSession
10. deregisterEnvironment
```

#### v2 (env-less) 流程

```
1. createCodeSession → cse_* session_id
2. fetchRemoteCredentials (POST /bridge) → worker_jwt, expires_in, api_base_url, worker_epoch
3. createV2ReplTransport (SSE + CCRClient)
4. connect SSE stream
5. [运行会话...]
6. 定期JWT刷新 (5分钟缓冲)
7. archiveSession (POST /v1/sessions/{id}/archive)
```

### 3.2 核心实现分析

#### `remoteBridgeCore.ts` - v2实现

这是env-less路径的核心实现，约1000行代码。

**初始化流程** (`initEnvLessBridgeCore`):

```typescript
export async function initEnvLessBridgeCore(
  params: EnvLessBridgeParams,
): Promise<ReplBridgeHandle | null> {
  
  // ── 1. 创建会话 ──────────────────────────────────────
  const sessionId = await withRetry(
    () => createCodeSession(baseUrl, accessToken, title, cfg.http_timeout_ms, tags),
    'createCodeSession',
    cfg,
  )
  
  // ── 2. 获取Bridge凭证 ──────────────────────────────────
  const credentials = await withRetry(
    () => fetchRemoteCredentials(sessionId, baseUrl, accessToken, cfg.http_timeout_ms),
    'fetchRemoteCredentials',
    cfg,
  )
  // credentials包含: worker_jwt, api_base_url, expires_in, worker_epoch
  
  // ── 3. 构建v2传输层 ────────────────────────────────────
  const transport = await createV2ReplTransport({
    sessionUrl: buildCCRv2SdkUrl(credentials.api_base_url, sessionId),
    ingressToken: credentials.worker_jwt,
    sessionId,
    epoch: credentials.worker_epoch,
    heartbeatIntervalMs: cfg.heartbeat_interval_ms,
    heartbeatJitterFraction: cfg.heartbeat_jitter_fraction,
    getAuthToken: () => credentials.worker_jwt,
    outboundOnly,
  })
  
  // ── 4. 状态初始化 ──────────────────────────────────────
  // Echo去重：消息UUID环形缓冲区
  const recentPostedUUIDs = new BoundedUUIDSet(cfg.uuid_dedup_buffer_size)
  const recentInboundUUIDs = new BoundedUUIDSet(cfg.uuid_dedup_buffer_size)
  
  // 初始消息UUID集合
  const initialMessageUUIDs = new Set<string>()
  if (initialMessages) {
    for (const msg of initialMessages) {
      initialMessageUUIDs.add(msg.uuid)
      recentPostedUUIDs.add(msg.uuid)
    }
  }
  
  // FlushGate：队列刷新期间的消息
  const flushGate = new FlushGate<Message>()
  
  // ── 5. JWT刷新调度器 ──────────────────────────────────
  const refresh = createTokenRefreshScheduler({
    refreshBufferMs: cfg.token_refresh_buffer_ms, // 默认5分钟
    getAccessToken: async () => {
      const stale = getAccessToken()
      if (onAuth401) await onAuth401(stale ?? '')
      return getAccessToken() ?? stale
    },
    onRefresh: (sid, oauthToken) => {
      // 重新获取凭证并重建传输层
      void rebuildTransport(fresh, 'proactive_refresh')
    },
  })
  refresh.scheduleFromExpiresIn(sessionId, credentials.expires_in)
  
  // ── 6. 连接传输层 ─────────────────────────────────────
  wireTransportCallbacks()
  transport.connect()
  
  // ── 7. 返回句柄 ───────────────────────────────────────
  return {
    bridgeSessionId: sessionId,
    environmentId: '',
    sessionIngressUrl: credentials.api_base_url,
    writeMessages(messages) { ... },
    writeSdkMessages(messages) { ... },
    sendControlRequest(request) { ... },
    sendControlResponse(response) { ... },
    sendControlCancelRequest(requestId) { ... },
    sendResult() { ... },
    teardown() { ... },
  }
}
```

**关键设计点**：

1. **重试机制** - 使用指数退避重试初始化步骤：
```typescript
async function withRetry<T>(
  fn: () => Promise<T | null>,
  label: string,
  cfg: EnvLessBridgeConfig,
): Promise<T | null> {
  const max = cfg.init_retry_max_attempts // 默认3次
  for (let attempt = 1; attempt <= max; attempt++) {
    const result = await fn()
    if (result !== null) return result
    if (attempt < max) {
      const base = cfg.init_retry_base_delay_ms * 2 ** (attempt - 1)
      const jitter = base * cfg.init_retry_jitter_fraction * (2 * Math.random() - 1)
      const delay = Math.min(base + jitter, cfg.init_retry_max_delay_ms)
      await sleep(delay)
    }
  }
  return null
}
```

2. **传输层重建** - JWT过期或epoch冲突时重建：
```typescript
async function rebuildTransport(
  fresh: RemoteCredentials,
  cause: 'proactive_refresh' | 'auth_401_recovery',
): Promise<void> {
  // 防止写入丢失：启动flushGate
  flushGate.start()
  
  try {
    // 保存序列号（防止历史重放）
    const seq = transport.getLastSequenceNum()
    transport.close()
    
    // 创建新传输层
    transport = await createV2ReplTransport({
      sessionUrl: buildCCRv2SdkUrl(fresh.api_base_url, sessionId),
      ingressToken: fresh.worker_jwt,
      sessionId,
      epoch: fresh.worker_epoch,
      initialSequenceNum: seq,  // 关键：恢复序列号
      getAuthToken: () => fresh.worker_jwt,
    })
    
    // 重连回调
    wireTransportCallbacks()
    transport.connect()
    
    // 刷新JWT调度
    refresh.scheduleFromExpiresIn(sessionId, fresh.expires_in)
    
    // 排队消息
    drainFlushGate()
  } finally {
    flushGate.drop()
  }
}
```

3. **401错误恢复** - SSE收到401时刷新OAuth：
```typescript
async function recoverFromAuthFailure(): Promise<void> {
  if (authRecoveryInFlight) return
  authRecoveryInFlight = true
  onStateChange?.('reconnecting', 'JWT expired — refreshing')
  
  try {
    // 刷新OAuth令牌
    const stale = getAccessToken()
    if (onAuth401) await onAuth401(stale ?? '')
    const oauthToken = getAccessToken() ?? stale
    
    // 重新获取凭证
    const fresh = await withRetry(
      () => fetchRemoteCredentials(sessionId, baseUrl, oauthToken, cfg.http_timeout_ms),
      'fetchRemoteCredentials (recovery)',
      cfg,
    )
    
    // 重置初始刷新标志
    initialFlushDone = false
    
    // 重建传输层
    await rebuildTransport(fresh, 'auth_401_recovery')
  } finally {
    authRecoveryInFlight = false
  }
}
```

### 3.3 会话创建API

#### `codeSessionApi.ts` - v2会话API

```typescript
// 创建会话
export async function createCodeSession(
  baseUrl: string,
  accessToken: string,
  title: string,
  timeoutMs: number,
  tags?: string[],
): Promise<string | null> {
  const url = `${baseUrl}/v1/code/sessions`
  const response = await axios.post(
    url,
    { title, bridge: {}, ...(tags?.length ? { tags } : {}) },
    {
      headers: oauthHeaders(accessToken),
      timeout: timeoutMs,
      validateStatus: s => s < 500,
    },
  )
  // 返回 cse_* 格式的session ID
  return response.data.session.id
}

// 获取Bridge凭证
export async function fetchRemoteCredentials(
  sessionId: string,
  baseUrl: string,
  accessToken: string,
  timeoutMs: number,
  trustedDeviceToken?: string,
): Promise<RemoteCredentials | null> {
  const url = `${baseUrl}/v1/code/sessions/${sessionId}/bridge`
  const headers = oauthHeaders(accessToken)
  if (trustedDeviceToken) {
    headers['X-Trusted-Device-Token'] = trustedDeviceToken
  }
  const response = await axios.post(url, {}, { headers, timeout: timeoutMs })
  return {
    worker_jwt: response.data.worker_jwt,
    api_base_url: response.data.api_base_url,
    expires_in: response.data.expires_in,
    worker_epoch: response.data.worker_epoch,
  }
}
```

#### `createSession.ts` - v1会话API

```typescript
export async function createBridgeSession({
  environmentId,
  title,
  events,
  gitRepoUrl,
  branch,
}: {...}): Promise<string | null> {
  // 构建git源信息
  let gitSource: GitSource | null = null
  if (gitRepoUrl) {
    gitSource = {
      type: 'git_repository',
      url: `https://${host}/${owner}/${name}`,
      revision: branch,
    }
  }
  
  // POST /v1/sessions
  const requestBody = {
    title,
    events,
    session_context: {
      sources: gitSource ? [gitSource] : [],
      outcomes: [...],
      model: getMainLoopModel(),
    },
    environment_id: environmentId,
    source: 'remote-control',
  }
  
  const response = await axios.post(`${baseUrl}/v1/sessions`, requestBody, {
    headers: {
      ...getOAuthHeaders(accessToken),
      'anthropic-beta': 'ccr-byoc-2025-07-29',
      'x-organization-uuid': orgUUID,
    },
  })
  
  return response.data.id // session_* 格式
}
```

---

## 4. 消息传递机制

### 4.1 传输层抽象

#### `replBridgeTransport.ts`

定义统一的传输层接口，适配v1和v2：

```typescript
export type ReplBridgeTransport = {
  write(message: StdoutMessage): Promise<void>
  writeBatch(messages: StdoutMessage[]): Promise<void>
  close(): void
  isConnectedStatus(): boolean
  getStateLabel(): string
  setOnData(callback: (data: string) => void): void
  setOnClose(callback: (closeCode?: number) => void): void
  setOnConnect(callback: () => void): void
  connect(): void
  getLastSequenceNum(): number    // SSE序列号（防止历史重放）
  droppedBatchCount: number       // 丢弃批次计数
  reportState(state: SessionState): void  // worker状态报告
  reportMetadata(metadata: Record<string, unknown>): void
  reportDelivery(eventId: string, status: 'processing' | 'processed'): void
  flush(): Promise<void>
}
```

#### v1适配器

```typescript
export function createV1ReplTransport(
  hybrid: HybridTransport,
): ReplBridgeTransport {
  return {
    write: msg => hybrid.write(msg),
    writeBatch: msgs => hybrid.writeBatch(msgs),
    close: () => hybrid.close(),
    // ...其他适配
    getLastSequenceNum: () => 0,  // v1不使用SSE序列号
    reportState: () => {},        // v1不支持状态报告
  }
}
```

#### v2适配器

```typescript
export async function createV2ReplTransport(opts: {
  sessionUrl: string
  ingressToken: string
  sessionId: string
  epoch?: number
  heartbeatIntervalMs?: number
  outboundOnly?: boolean
  getAuthToken?: () => string | undefined
}): Promise<ReplBridgeTransport> {
  
  // 注册worker（获取epoch）
  const epoch = opts.epoch ?? (await registerWorker(sessionUrl, ingressToken))
  
  // SSE传输（读取）
  const sseUrl = new URL(sessionUrl)
  sseUrl.pathname += '/worker/events/stream'
  const sse = new SSETransport(sseUrl, {}, sessionId, undefined, initialSequenceNum, getAuthHeaders)
  
  // CCR客户端（写入+心跳）
  const ccr = new CCRClient(sse, new URL(sessionUrl), {
    getAuthHeaders,
    heartbeatIntervalMs: opts.heartbeatIntervalMs,
    onEpochMismatch: () => {
      ccr.close()
      sse.close()
      onCloseCb?.(4090)  // epoch冲突关闭码
    },
  })
  
  return {
    write(msg) { return ccr.writeEvent(msg) },
    writeBatch(msgs) { ... },
    close() { ccr.close(); sse.close() },
    getLastSequenceNum() { return sse.getLastSequenceNum() },
    reportState(state) { ccr.reportState(state) },
    connect() {
      if (!opts.outboundOnly) void sse.connect()
      void ccr.initialize(epoch).then(
        () => { ccrInitialized = true; onConnectCb?.() },
        (err) => { onCloseCb?.(4091) },  // 初始化失败
      )
    },
  }
}
```

### 4.2 消息处理

#### `bridgeMessaging.ts` - 消息路由

```typescript
// 入站消息处理
export function handleIngressMessage(
  data: string,
  recentPostedUUIDs: BoundedUUIDSet,
  recentInboundUUIDs: BoundedUUIDSet,
  onInboundMessage?: (msg: SDKMessage) => void,
  onPermissionResponse?: (response: SDKControlResponse) => void,
  onControlRequest?: (request: SDKControlRequest) => void,
): void {
  const parsed = jsonParse(data)
  
  // 1. 权限响应
  if (isSDKControlResponse(parsed)) {
    onPermissionResponse?.(parsed)
    return
  }
  
  // 2. 控制请求
  if (isSDKControlRequest(parsed)) {
    onControlRequest?.(parsed)
    return
  }
  
  // 3. SDK消息
  if (!isSDKMessage(parsed)) return
  
  // 4. Echo去重
  const uuid = parsed.uuid
  if (uuid && recentPostedUUIDs.has(uuid)) {
    logForDebugging(`[bridge:repl] Ignoring echo: uuid=${uuid}`)
    return
  }
  
  // 5. 重投去重
  if (uuid && recentInboundUUIDs.has(uuid)) {
    logForDebugging(`[bridge:repl] Ignoring re-delivered inbound: uuid=${uuid}`)
    return
  }
  
  // 6. 用户消息转发
  if (parsed.type === 'user') {
    if (uuid) recentInboundUUIDs.add(uuid)
    void onInboundMessage?.(parsed)
  }
}

// 服务器控制请求处理
export function handleServerControlRequest(
  request: SDKControlRequest,
  handlers: ServerControlRequestHandlers,
): void {
  switch (request.request.subtype) {
    case 'initialize':
      // 返回能力信息
      response = {
        type: 'control_response',
        response: {
          subtype: 'success',
          request_id: request.request_id,
          response: { commands: [], output_style: 'normal', models: [], pid: process.pid },
        },
      }
      break
    
    case 'set_model':
      handlers.onSetModel?.(request.request.model)
      break
    
    case 'interrupt':
      handlers.onInterrupt?.()
      break
    
    case 'set_permission_mode':
      const verdict = handlers.onSetPermissionMode?.(request.request.mode)
      // 返回结果或错误
      break
    
    case 'can_use_tool':
      // 权限提示请求
      handlers.transport.reportState('requires_action')
      break
  }
  
  handlers.transport.write(response)
}
```

### 4.3 消息类型

```typescript
// SDK消息类型
type SDKMessage = 
  | { type: 'user', message: {...}, uuid: string }
  | { type: 'assistant', message: {...}, uuid: string }
  | { type: 'system', subtype: 'local_command', ... }

// 控制请求类型
type SDKControlRequest = {
  type: 'control_request'
  request_id: string
  request: {
    subtype: 'initialize' | 'set_model' | 'interrupt' | 
             'set_permission_mode' | 'can_use_tool' | 'set_max_thinking_tokens'
    ...
  }
}

// 控制响应类型
type SDKControlResponse = {
  type: 'control_response'
  response: {
    subtype: 'success' | 'error'
    request_id: string
    response?: Record<string, unknown>
    error?: string
  }
}

// 结果消息（会话结束）
type SDKResultSuccess = {
  type: 'result'
  subtype: 'success'
  duration_ms: number
  usage: Usage
  session_id: string
  uuid: string
}
```

### 4.4 FlushGate - 刷新门控

防止初始历史刷新与新消息交错：

```typescript
export class FlushGate<T> {
  private _active = false
  private _pending: T[] = []
  
  // 启动门控
  start(): void {
    this._active = true
  }
  
  // 入队（如果active）
  enqueue(...items: T[]): boolean {
    if (!this._active) return false
    this._pending.push(...items)
    return true
  }
  
  // 结束并返回排队的消息
  end(): T[] {
    this._active = false
    return this._pending.splice(0)
  }
  
  // 丢弃（传输层关闭）
  drop(): number {
    this._active = false
    const count = this._pending.length
    this._pending.length = 0
    return count
  }
}
```

---

## 5. 安全机制

### 5.1 OAuth认证

Bridge需要claude.ai订阅者的OAuth令牌：

```typescript
// bridgeConfig.ts
export function getBridgeAccessToken(): string | undefined {
  return getBridgeTokenOverride() ?? getClaudeAIOAuthTokens()?.accessToken
}

export function getBridgeBaseUrl(): string {
  return getBridgeBaseUrlOverride() ?? getOauthConfig().BASE_API_URL
}
```

**认证流程**：
1. `/login` 获取OAuth令牌（需user:profile scope）
2. 检查 `isClaudeAISubscriber()` 排除Bedrock/Vertex
3. 检查GrowthBook开关 `tengu_ccr_bridge`
4. 检查组织策略 `allow_remote_control`

### 5.2 JWT刷新调度

自动刷新即将过期的JWT：

```typescript
// jwtUtils.ts
export function createTokenRefreshScheduler({
  getAccessToken,
  onRefresh,
  refreshBufferMs = 5 * 60 * 1000,  // 5分钟缓冲
}): {...} {
  
  const timers = new Map<string, ReturnType<typeof setTimeout>>()
  const generations = new Map<string, number>()  // 防止过期刷新
  
  function schedule(sessionId: string, token: string): void {
    const expiry = decodeJwtExpiry(token)
    const delayMs = expiry * 1000 - Date.now() - refreshBufferMs
    
    if (delayMs <= 0) {
      void doRefresh(sessionId, gen)  // 立即刷新
      return
    }
    
    const timer = setTimeout(doRefresh, delayMs, sessionId, gen)
    timers.set(sessionId, timer)
  }
  
  // 使用expires_in调度（不解码JWT）
  function scheduleFromExpiresIn(sessionId: string, expiresInSeconds: number): void {
    const delayMs = Math.max(expiresInSeconds * 1000 - refreshBufferMs, 30_000)
    const timer = setTimeout(doRefresh, delayMs, sessionId, gen)
    timers.set(sessionId, timer)
  }
  
  async function doRefresh(sessionId: string, gen: number): Promise<void> {
    const oauthToken = await getAccessToken()
    
    // 检查generation防止过期
    if (generations.get(sessionId) !== gen) return
    
    if (!oauthToken) {
      // 重试逻辑
      if (failures < MAX_REFRESH_FAILURES) {
        const retryTimer = setTimeout(doRefresh, 60_000, sessionId, gen)
        timers.set(sessionId, retryTimer)
      }
      return
    }
    
    onRefresh(sessionId, oauthToken)
    
    // 后续刷新
    const timer = setTimeout(doRefresh, 30 * 60 * 1000, sessionId, gen)
    timers.set(sessionId, timer)
  }
  
  return { schedule, scheduleFromExpiresIn, cancel, cancelAll }
}
```

### 5.3 信任设备令牌

用于高安全等级会话：

```typescript
// trustedDevice.ts
const TRUSTED_DEVICE_GATE = 'tengu_sessions_elevated_auth_enforcement'

export function getTrustedDeviceToken(): string | undefined {
  if (!isGateEnabled()) return undefined
  return readStoredToken()  // 从keychain读取
}

// 登录时注册设备
export async function enrollTrustedDevice(): Promise<void> {
  if (!(await checkGate_CACHED_OR_BLOCKING(TRUSTED_DEVICE_GATE))) return
  
  const response = await axios.post(
    `${baseUrl}/api/auth/trusted_devices`,
    { display_name: `Claude Code on ${hostname()} · ${process.platform}` },
    { headers: { Authorization: `Bearer ${accessToken}` } },
  )
  
  const token = response.data?.device_token
  secureStorage.update({ ...storageData, trustedDeviceToken: token })
}
```

### 5.4 权限控制

```typescript
// bridgeEnabled.ts
export async function getBridgeDisabledReason(): Promise<string | null> {
  if (!isClaudeAISubscriber()) {
    return 'Remote Control requires a claude.ai subscription. Run `claude auth login`...'
  }
  if (!hasProfileScope()) {
    return 'Remote Control requires a full-scope login token...'
  }
  if (!getOauthAccountInfo()?.organizationUuid) {
    return 'Unable to determine your organization...'
  }
  if (!(await checkGate_CACHED_OR_BLOCKING('tengu_ccr_bridge'))) {
    return 'Remote Control is not yet enabled for your account.'
  }
  return null
}
```

---

## 6. 用户使用指南

### 6.1 启用Remote Control

**前提条件**：
- claude.ai Pro/Team订阅
- 使用OAuth登录（不是API key）
- 组织允许远程控制

**启用方式**：

1. **自动启用**（配置）：
```bash
# 配置自动启动
claude config set remoteControlAtStartup true

# 每次会话自动连接
claude
# 显示：Remote Control connected (session_abc123)
```

2. **手动启用**：
```bash
claude
> /remote-control my-session
# 或简写
> /rc my-session
```

3. **独立模式**：
```bash
claude remote-control --continue
# 等待远程会话连接
```

### 6.2 使用claude.ai

**访问方式**：
- 网页：https://claude.ai/code
- 移动应用：iOS/Android Claude app

**会话列表显示**：
- 本地终端的会话会出现在列表中
- 显示会话标题、状态、活动
- 可实时查看输出和工具执行

**交互操作**：
- 发送消息
- 批准/拒绝权限请求
- 中断执行
- 切换模型
- 重命名会话

### 6.3 断开连接

```bash
# REPL中断开
> /remote-control
# 显示：Remote Control disconnected.

# 或退出会话
> /exit
```

### 6.4 故障排查

**常见问题**：

1. **"Remote Control is not yet enabled for your account"**
   - 需要claude.ai订阅
   - 功能逐步 rollout

2. **"Remote Control requires a full-scope login token"**
   - 使用 `claude auth login` 重新登录
   - 不支持 `setup-token` 或环境变量token

3. **连接断开**
   - 检查网络连接
   - OAuth令牌可能过期，运行 `/login` 刷新

4. **会话不显示**
   - 确保Remote Control已连接
   - 检查会话是否已归档

---

## 7. 从零实现Bridge

### 7.1 最小实现框架

```typescript
// minimal-bridge.ts
import { createCodeSession, fetchRemoteCredentials } from './codeSessionApi.js'
import { createV2ReplTransport } from './replBridgeTransport.js'

export async function createMinimalBridge(
  accessToken: string,
  baseUrl: string,
): Promise<{ send: (msg: any) => void, close: () => void }> {
  
  // 1. 创建会话
  const sessionId = await createCodeSession(baseUrl, accessToken, 'My Session', 10000)
  if (!sessionId) throw new Error('Failed to create session')
  
  // 2. 获取凭证
  const creds = await fetchRemoteCredentials(sessionId, baseUrl, accessToken, 10000)
  if (!creds) throw new Error('Failed to fetch credentials')
  
  // 3. 创建传输层
  const transport = await createV2ReplTransport({
    sessionUrl: `${creds.api_base_url}/v1/code/sessions/${sessionId}`,
    ingressToken: creds.worker_jwt,
    sessionId,
    epoch: creds.worker_epoch,
  })
  
  // 4. 设置消息处理
  transport.setOnData((data: string) => {
    const msg = JSON.parse(data)
    if (msg.type === 'user') {
      console.log('收到消息:', msg.message.content)
    }
  })
  
  transport.setOnConnect(() => {
    console.log('Bridge已连接')
  })
  
  // 5. 连接
  transport.connect()
  
  // 6. 返回接口
  return {
    send: (msg: any) => transport.write(msg),
    close: () => transport.close(),
  }
}
```

### 7.2 完整实现步骤

#### 步骤1：认证层

```typescript
// auth-layer.ts
import { getClaudeAIOAuthTokens } from './utils/auth.js'
import { getOauthConfig } from './constants/oauth.js'

export async function checkAuth(): Promise<{ accessToken: string; baseUrl: string }> {
  const tokens = getClaudeAIOAuthTokens()
  if (!tokens?.accessToken) {
    throw new Error('请先登录: claude auth login')
  }
  
  // 检查订阅类型
  if (!tokens.refreshToken) {
    throw new Error('需要claude.ai订阅')
  }
  
  return {
    accessToken: tokens.accessToken,
    baseUrl: getOauthConfig().BASE_API_URL,
  }
}
```

#### 步骤2：会话创建

```typescript
// session-layer.ts
export async function createBridgeSession(
  accessToken: string,
  baseUrl: string,
  title: string,
): Promise<{ sessionId: string; workerJwt: string; apiUrl: string }> {
  
  // POST /v1/code/sessions
  const sessionResp = await axios.post(
    `${baseUrl}/v1/code/sessions`,
    { title, bridge: {} },
    {
      headers: {
        Authorization: `Bearer ${accessToken}`,
        'anthropic-version': '2023-06-01',
      },
    },
  )
  const sessionId = sessionResp.data.session.id  // cse_*
  
  // POST /bridge
  const bridgeResp = await axios.post(
    `${baseUrl}/v1/code/sessions/${sessionId}/bridge`,
    {},
    {
      headers: {
        Authorization: `Bearer ${accessToken}`,
        'anthropic-version': '2023-06-01',
      },
    },
  )
  
  return {
    sessionId,
    workerJwt: bridgeResp.data.worker_jwt,
    apiUrl: bridgeResp.data.api_base_url,
  }
}
```

#### 步骤3：传输层

```typescript
// transport-layer.ts
export class BridgeTransport {
  private sse: SSETransport
  private ccr: CCRClient
  
  constructor(
    sessionUrl: string,
    workerJwt: string,
    sessionId: string,
    epoch: number,
  ) {
    // SSE读取流
    this.sse = new SSETransport(
      new URL(`${sessionUrl}/worker/events/stream`),
      {},
      sessionId,
    )
    
    // CCR写入客户端
    this.ccr = new CCRClient(this.sse, new URL(sessionUrl), {
      getAuthHeaders: () => ({ Authorization: `Bearer ${workerJwt}` }),
    })
  }
  
  connect() {
    void this.sse.connect()
    void this.ccr.initialize(this.epoch)
  }
  
  send(message: any) {
    return this.ccr.writeEvent(message)
  }
  
  onMessage(callback: (data: string) => void) {
    this.sse.setOnData(callback)
  }
  
  close() {
    this.ccr.close()
    this.sse.close()
  }
}
```

#### 步骤4：消息处理

```typescript
// message-handler.ts
export class MessageHandler {
  private sentUUIDs = new Set<string>()
  private receivedUUIDs = new Set<string>()
  
  handleInbound(data: string, onMessage: (msg: SDKMessage) => void): void {
    const parsed = JSON.parse(data)
    
    // Echo去重
    if (parsed.uuid && this.sentUUIDs.has(parsed.uuid)) return
    
    // 重投去重
    if (parsed.uuid && this.receivedUUIDs.has(parsed.uuid)) return
    
    if (parsed.type === 'user') {
      if (parsed.uuid) this.receivedUUIDs.add(parsed.uuid)
      onMessage(parsed)
    }
  }
  
  prepareOutbound(messages: Message[]): SDKMessage[] {
    const events = messages.map(m => {
      if (m.uuid) this.sentUUIDs.add(m.uuid)
      return { ...m, session_id: this.sessionId }
    })
    return events
  }
}
```

#### 步骤5：集成

```typescript
// bridge-integration.ts
export class Bridge {
  private transport: BridgeTransport
  private handler: MessageHandler
  private refreshScheduler: TokenRefreshScheduler
  
  async start() {
    const { accessToken, baseUrl } = await checkAuth()
    const { sessionId, workerJwt, apiUrl } = await createBridgeSession(
      accessToken, baseUrl, 'My Bridge'
    )
    
    this.transport = new BridgeTransport(
      `${apiUrl}/v1/code/sessions/${sessionId}`,
      workerJwt, sessionId, epoch,
    )
    
    this.handler = new MessageHandler(sessionId)
    
    this.transport.onMessage((data) => {
      this.handler.handleInbound(data, (msg) => {
        this.processInboundMessage(msg)
      })
    })
    
    this.transport.connect()
    
    // JWT刷新
    this.refreshScheduler = createTokenRefreshScheduler({
      getAccessToken: () => accessToken,
      onRefresh: async () => {
        const newCreds = await fetchRemoteCredentials(sessionId, baseUrl, accessToken, 10000)
        this.transport.rebuild(newCreds.workerJwt, newCreds.worker_epoch)
      },
    })
    this.refreshScheduler.scheduleFromExpiresIn(sessionId, expires_in)
  }
  
  sendMessages(messages: Message[]) {
    const events = this.handler.prepareOutbound(messages)
    this.transport.send(events)
  }
  
  async close() {
    this.refreshScheduler.cancelAll()
    this.transport.close()
    
    // 归档会话
    await axios.post(`${baseUrl}/v1/sessions/${sessionId}/archive`)
  }
}
```

### 7.3 关键设计要点

1. **Echo去重**：WebSocket会回显发送的消息，需要识别并忽略
2. **序列号恢复**：传输层重建时保留序列号，防止历史重放
3. **JWT刷新**：提前刷新（5分钟缓冲），防止过期中断
4. **错误恢复**：401时刷新OAuth，epoch冲突时重建传输层
5. **状态报告**：定期心跳，报告worker状态
6. **优雅关闭**：发送result消息、归档会话

---

## 附录：配置参数

### env-less配置

```typescript
export const DEFAULT_ENV_LESS_BRIDGE_CONFIG = {
  init_retry_max_attempts: 3,            // 初始化重试次数
  init_retry_base_delay_ms: 500,         // 重试基础延迟
  init_retry_jitter_fraction: 0.25,      // 重试抖动因子
  init_retry_max_delay_ms: 4000,         // 最大重试延迟
  http_timeout_ms: 10_000,               // HTTP超时
  uuid_dedup_buffer_size: 2000,          // UUID去重缓冲
  heartbeat_interval_ms: 20_000,         // 心跳间隔
  heartbeat_jitter_fraction: 0.1,        // 心跳抖动
  token_refresh_buffer_ms: 300_000,      // JWT刷新缓冲(5分钟)
  teardown_archive_timeout_ms: 1500,     // 关闭超时
  connect_timeout_ms: 15_000,            // 连接超时
  min_version: '0.0.0',                  // 最低版本
}
```

### GrowthBook开关

| 开关名称 | 作用 |
|---------|------|
| `tengu_ccr_bridge` | 启用Bridge功能 |
| `tengu_bridge_repl_v2` | 使用v2 env-less路径 |
| `tengu_bridge_repl_v2_config` | v2配置参数 |
| `tengu_sessions_elevated_auth_enforcement` | 信任设备令牌 |
| `tengu_bridge_initial_history_cap` | 初始历史消息上限 |
| `tengu_cobalt_harbor` | 自动连接默认 |

---

## 总结

Bridge远程控制模式是Claude Code CLI的重要功能，实现了本地终端与云端平台的实时双向通信。通过深入分析源码，我们理解了：

1. **两种架构路径**：v1环境轮询模式、v2直接会话模式
2. **消息传递机制**：WebSocket/SSE传输、去重、控制协议
3. **安全认证体系**：OAuth、JWT刷新、信任设备令牌
4. **会话生命周期**：创建、连接、刷新、归档
5. **错误恢复策略**：401刷新、epoch重建、序列号恢复

这些设计保证了Bridge功能的可靠性、安全性和用户体验。