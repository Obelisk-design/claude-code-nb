# LSP 语言服务器集成

**分析日期:** 2026-04-02  
**源码路径:** `services/lsp/`  
**文件清单:** 7个核心文件

---

## 目录

1. [LSP协议概述](#1-lsp协议概述)
2. [LSPClient.ts 分析](#2-lspclientts-分析)
3. [LSPServerInstance.ts 分析](#3-lspserverinstancets-分析)
4. [LSPServerManager.ts 分析](#4-lspservermanagerts-分析)
5. [语言服务器配置](#5-语言服务器配置)
6. [诊断信息处理](#6-诊断信息处理)
7. [用户使用指南](#7-用户使用指南)
8. [从零实现](#8-从零实现)

---

## 1. LSP协议概述

### 1.1 什么是LSP

**Language Server Protocol (LSP)** 是由微软开发的标准协议，用于编辑器与语言服务器之间的通信。它定义了一套JSON-RPC协议规范，使编辑器可以获得代码补全、错误诊断、跳转定义等智能功能。

**核心优势：**
- **语言无关性**: 同一个编辑器可支持多种语言
- **编辑器无关性**: 同一个语言服务器可服务于多个编辑器
- **标准化通信**: 使用JSON-RPC 2.0协议

### 1.2 LSP通信模式

LSP采用**JSON-RPC 2.0**协议，支持三种消息类型：

```typescript
// 请求(Request) - 需要响应
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "textDocument/definition",
  "params": { ... }
}

// 响应(Response) - 回复请求
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": { ... }
}

// 通知(Notification) - 无需响应
{
  "jsonrpc": "2.0",
  "method": "textDocument/publishDiagnostics",
  "params": { ... }
}
```

### 1.3 LSP生命周期

```
┌─────────────────────────────────────────────────────────────┐
│                    LSP Server生命周期                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. 启动阶段                                                  │
│     ┌──────────┐                                             │
│     │  spawn   │  启动子进程                                  │
│     │ process  │                                             │
│     └───┬──────┘                                             │
│         │                                                    │
│         v                                                    │
│  2. 初始化阶段                                                │
│     ┌──────────┐                                             │
│     │initialize│  exchange capabilities                      │
│     │ request  │                                             │
│     └───┬──────┘                                             │
│         │                                                    │
│         v                                                    │
│     ┌────────────┐                                           │
│     │initialized │  客户端就绪通知                            │
│     │notification│                                           │
│     └───┬────────┘                                           │
│         │                                                    │
│         v                                                    │
│  3. 运行阶段                                                  │
│     ┌──────────┐    ┌──────────┐    ┌──────────┐            │
│     │ didOpen  │───>│didChange │───>│ didSave  │            │
│     │          │    │          │    │          │            │
│     └───┬──────┘    └───┬──────┘    └───┬──────┘            │
│         │               │               │                   │
│         v               v               v                   │
│     ┌────────────────────────────────────────┐              │
│     │  textDocument/publishDiagnostics        │              │
│     │  (服务器主动推送诊断信息)                │              │
│     └────────────────────────────────────────┘              │
│                                                              │
│  4. 关闭阶段                                                  │
│     ┌──────────┐                                             │
│     │ shutdown │  请求关闭                                   │
│     │ request  │                                             │
│     └───┬──────┘                                             │
│         │                                                    │
│         v                                                    │
│     ┌──────────┐                                             │
│     │   exit   │  退出通知                                   │
│     │notification│                                           │
│     └───┬──────┘                                             │
│         │                                                    │
│         v                                                    │
│     ┌──────────┐                                             │
│     │  kill    │  终止进程                                   │
│     │ process  │                                             │
│     └──────────┘                                             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 1.4 核心LSP方法

| 方法 | 类型 | 功能 |
|------|------|------|
| `initialize` | Request | 初始化服务器，交换能力 |
| `initialized` | Notification | 客户端就绪通知 |
| `shutdown` | Request | 关闭服务器 |
| `exit` | Notification | 退出进程 |
| `textDocument/didOpen` | Notification | 文件打开通知 |
| `textDocument/didChange` | Notification | 文件变更通知 |
| `textDocument/didSave` | Notification | 文件保存通知 |
| `textDocument/didClose` | Notification | 文件关闭通知 |
| `textDocument/publishDiagnostics` | Notification | 发布诊断信息 |
| `textDocument/definition` | Request | 跳转定义 |
| `textDocument/hover` | Request | 获取悬停信息 |
| `textDocument/references` | Request | 查找引用 |
| `workspace/configuration` | Request | 服务器请求配置 |

---

## 2. LSPClient.ts 分析

**文件路径:** `services/lsp/LSPClient.ts`  
**核心职责:** 底层LSP通信客户端，管理进程与JSON-RPC连接

### 2.1 模块结构

```typescript
// 导入依赖
import { spawn, type ChildProcess } from 'child_process'
import {
  createMessageConnection,
  StreamMessageReader,
  StreamMessageWriter,
  Trace,
} from 'vscode-jsonrpc/node.js'

// 导出的主要类型
export type LSPClient = {
  readonly capabilities: ServerCapabilities | undefined
  readonly isInitialized: boolean
  start(command, args, options): Promise<void>
  initialize(params): Promise<InitializeResult>
  sendRequest<TResult>(method, params): Promise<TResult>
  sendNotification(method, params): Promise<void>
  onNotification(method, handler): void
  onRequest<TParams, TResult>(method, handler): void
  stop(): Promise<void>
}

// 主要工厂函数
export function createLSPClient(serverName, onCrash?): LSPClient
```

### 2.2 createLSPClient 工厂函数

采用**闭包模式**而非Class，实现状态封装：

```typescript
export function createLSPClient(
  serverName: string,
  onCrash?: (error: Error) => void,
): LSPClient {
  // 私有状态（闭包封装）
  let process: ChildProcess | undefined
  let connection: MessageConnection | undefined
  let capabilities: ServerCapabilities | undefined
  let isInitialized = false
  let startFailed = false
  let startError: Error | undefined
  let isStopping = false
  
  // 处理器队列（支持延迟注册）
  const pendingHandlers: Array<{
    method: string
    handler: (params: unknown) => void
  }> = []
  const pendingRequestHandlers: Array<{
    method: string
    handler: (params: unknown) => unknown | Promise<unknown>
  }> = []
  
  // 返回公共接口
  return {
    get capabilities() { return capabilities },
    get isInitialized() { return isInitialized },
    start, initialize, sendRequest, sendNotification,
    onNotification, onRequest, stop
  }
}
```

**设计要点：**
- **闭包状态**：所有私有变量通过闭包保护，外部无法直接访问
- **惰性初始化**：处理器可先注册，连接建立后自动应用
- **崩溃回调**：`onCrash`参数让上层管理器能响应进程崩溃

### 2.3 start() 方法详解

```typescript
async function start(
  command: string,
  args: string[],
  options?: { env?: Record<string, string>; cwd?: string }
): Promise<void> {
  try {
    // ========== 1. 启动子进程 ==========
    process = spawn(command, args, {
      stdio: ['pipe', 'pipe', 'pipe'],  // stdin/stdout/stderr都管道化
      env: { ...subprocessEnv(), ...options?.env },
      cwd: options?.cwd,
      windowsHide: true,  // Windows下隐藏控制台窗口
    })
    
    // ========== 1.5. 等待进程成功启动 ==========
    // 关键：spawn()立即返回，但错误事件异步触发
    // 必须等待'spawn'事件确认进程已启动
    const spawnedProcess = process
    await new Promise<void>((resolve, reject) => {
      const onSpawn = () => { cleanup(); resolve() }
      const onError = (error: Error) => { cleanup(); reject(error) }
      const cleanup = () => {
        spawnedProcess.removeListener('spawn', onSpawn)
        spawnedProcess.removeListener('error', onError)
      }
      spawnedProcess.once('spawn', onSpawn)
      spawnedProcess.once('error', onError)
    })
    
    // ========== 捕获stderr输出 ==========
    if (process.stderr) {
      process.stderr.on('data', (data: Buffer) => {
        const output = data.toString().trim()
        if (output) {
          logForDebugging(`[LSP SERVER ${serverName}] ${output}`)
        }
      })
    }
    
    // ========== 处理进程错误/退出 ==========
    process.on('error', error => {
      if (!isStopping) {
        startFailed = true
        startError = error
        logError(new Error(`LSP server ${serverName} failed to start: ${error.message}`))
      }
    })
    
    process.on('exit', (code, _signal) => {
      if (code !== 0 && code !== null && !isStopping) {
        isInitialized = false
        const crashError = new Error(`LSP server ${serverName} crashed with exit code ${code}`)
        logError(crashError)
        onCrash?.(crashError)  // 调用崩溃回调
      }
    })
    
    // ========== 2. 创建JSON-RPC连接 ==========
    const reader = new StreamMessageReader(process.stdout)
    const writer = new StreamMessageWriter(process.stdin)
    connection = createMessageConnection(reader, writer)
    
    // ========== 2.5. 注册连接错误处理器 ==========
    connection.onError(([error, _msg, _code]) => {
      if (!isStopping) {
        startFailed = true
        startError = error
        logError(new Error(`LSP server ${serverName} connection error: ${error.message}`))
      }
    })
    
    connection.onClose(() => {
      if (!isStopping) {
        isInitialized = false
        logForDebugging(`LSP server ${serverName} connection closed`)
      }
    })
    
    // ========== 3. 开始监听消息 ==========
    connection.listen()
    
    // ========== 3.5. 启用协议跟踪 ==========
    connection.trace(Trace.Verbose, {
      log: (message: string) => {
        logForDebugging(`[LSP PROTOCOL ${serverName}] ${message}`)
      }
    }).catch(error => {
      logForDebugging(`Failed to enable tracing: ${error.message}`)
    })
    
    // ========== 4. 应用延迟注册的处理器 ==========
    for (const { method, handler } of pendingHandlers) {
      connection.onNotification(method, handler)
    }
    pendingHandlers.length = 0
    
    for (const { method, handler } of pendingRequestHandlers) {
      connection.onRequest(method, handler)
    }
    pendingRequestHandlers.length = 0
    
  } catch (error) {
    logError(new Error(`LSP server ${serverName} failed to start: ${error.message}`))
    throw error
  }
}
```

**关键设计点：**

| 编号 | 设计要点 | 原因 |
|------|----------|------|
| 1.5 | 等待spawn事件 | spawn()异步返回，ENOENT等错误异步触发 |
| 2.5 | 先注册错误处理器再listen() | 防止未处理的Promise rejection |
| isStopping标志 | 区分主动关闭与意外崩溃 | 避免关闭时产生虚假错误日志 |
| pendingHandlers | 延迟注册队列 | 支持在连接建立前注册处理器 |

### 2.4 initialize() 方法

```typescript
async function initialize(params: InitializeParams): Promise<InitializeResult> {
  if (!connection) {
    throw new Error('LSP client not started')
  }
  
  checkStartFailed()  // 检查启动是否失败
  
  try {
    // 发送initialize请求
    const result = await connection.sendRequest('initialize', params)
    
    // 保存服务器能力
    capabilities = result.capabilities
    
    // 发送initialized通知（LSP协议要求）
    await connection.sendNotification('initialized', {})
    
    isInitialized = true
    return result
  } catch (error) {
    logError(new Error(`LSP server ${serverName} initialize failed: ${error.message}`))
    throw error
  }
}
```

### 2.5 sendRequest/sendNotification 方法

```typescript
async function sendRequest<TResult>(method: string, params: unknown): Promise<TResult> {
  if (!connection) throw new Error('LSP client not started')
  checkStartFailed()
  if (!isInitialized) throw new Error('LSP server not initialized')
  
  try {
    return await connection.sendRequest(method, params)
  } catch (error) {
    logError(new Error(`request ${method} failed: ${error.message}`))
    throw error
  }
}

async function sendNotification(method: string, params: unknown): Promise<void> {
  if (!connection) throw new Error('LSP client not started')
  checkStartFailed()
  
  try {
    await connection.sendNotification(method, params)
  } catch (error) {
    // 通知失败不抛出（fire-and-forget）
    logForDebugging(`Notification ${method} failed but continuing`)
  }
}
```

**差异：**
- **Request**: 必须等待响应，失败抛出
- **Notification**: 无需响应，失败仅日志

### 2.6 onNotification/onRequest 方法

```typescript
function onNotification(method: string, handler: (params: unknown) => void): void {
  if (!connection) {
    // 连接未建立，队列延迟注册
    pendingHandlers.push({ method, handler })
    return
  }
  
  checkStartFailed()
  connection.onNotification(method, handler)
}

function onRequest<TParams, TResult>(
  method: string,
  handler: (params: TParams) => TResult | Promise<TResult>
): void {
  if (!connection) {
    pendingRequestHandlers.push({ method, handler: handler as ... })
    return
  }
  
  checkStartFailed()
  connection.onRequest(method, handler)
}
```

**延迟注册机制：** 允许在`start()`之前注册处理器，连接建立后自动应用。

### 2.7 stop() 方法

```typescript
async function stop(): Promise<void> {
  let shutdownError: Error | undefined
  
  // 标记正在关闭，防止错误处理器产生虚假日志
  isStopping = true
  
  try {
    if (connection) {
      // 发送shutdown请求（LSP协议要求）
      await connection.sendRequest('shutdown', {})
      // 发送exit通知（LSP协议要求）
      await connection.sendNotification('exit', {})
    }
  } catch (error) {
    shutdownError = error
    // 继续清理，即使shutdown失败
  } finally {
    // ========== 清理连接 ==========
    if (connection) {
      try {
        connection.dispose()
      } catch (error) {
        logForDebugging(`Connection disposal failed: ${errorMessage(error)}`)
      }
      connection = undefined
    }
    
    // ========== 清理进程 ==========
    if (process) {
      // 移除所有监听器防止内存泄漏
      process.removeAllListeners('error')
      process.removeAllListeners('exit')
      if (process.stdin) process.stdin.removeAllListeners('error')
      if (process.stderr) process.stderr.removeAllListeners('data')
      
      try {
        process.kill()
      } catch (error) {
        logForDebugging(`Process kill failed (may already be dead)`)
      }
      process = undefined
    }
    
    // 重置状态
    isInitialized = false
    capabilities = undefined
    isStopping = false
    
    if (shutdownError) {
      startFailed = true
      startError = shutdownError
    }
    
    // 如果有shutdown错误，在清理完成后抛出
    if (shutdownError) throw shutdownError
  }
}
```

**关闭流程：**
```
shutdown请求 → exit通知 → dispose连接 → kill进程 → 清理监听器
```

---

## 3. LSPServerInstance.ts 分析

**文件路径:** `services/lsp/LSPServerInstance.ts`  
**核心职责:** 单个LSP服务器实例的生命周期管理

### 3.1 状态机设计

```typescript
// 状态定义（推测，types.ts文件缺失）
type LspServerState = 
  | 'stopped'    // 未启动
  | 'starting'   // 启动中
  | 'running'    // 正常运行
  | 'stopping'   // 停止中
  | 'error'      // 错误状态
```

**状态转换图：**
```
                    ┌─────────────┐
                    │   stopped   │
                    └──────┬──────┘
                           │ start()
                           v
                    ┌─────────────┐
                    │   starting  │
                    └──────┬──────┘
                           │ 成功
                           v
         ┌─────────────────────────────┐
         │         running             │◄─────── restart()
         └───────────────┬─────────────┘
                         │ stop() / 崩溃
                         v
                  ┌─────────────┐
                  │  stopping   │
                  └──────┬──────┘
                         │
         ┌───────────────┴───────────────┐
         │                               │
         v                               v
  ┌─────────────┐                 ┌─────────────┐
  │   stopped   │                 │    error    │
  └─────────────┘                 └──────┬──────┘
                                         │ 重试
                                         v
                                  ┌─────────────┐
                                  │   starting  │
                                  └─────────────┘
```

### 3.2 类型定义

```typescript
export type LSPServerInstance = {
  readonly name: string                    // 服务器唯一标识
  readonly config: ScopedLspServerConfig   // 配置对象
  readonly state: LspServerState           // 当前状态
  readonly startTime: Date | undefined     // 启动时间
  readonly lastError: Error | undefined    // 最后错误
  readonly restartCount: number            // 重启次数
  
  start(): Promise<void>                   // 启动服务器
  stop(): Promise<void>                    // 停止服务器
  restart(): Promise<void>                 // 手动重启
  isHealthy(): boolean                     // 健康检查
  sendRequest<T>(method, params): Promise<T>
  sendNotification(method, params): Promise<void>
  onNotification(method, handler): void
  onRequest<TParams, TResult>(method, handler): void
}
```

### 3.3 createLSPServerInstance 工厂函数

```typescript
export function createLSPServerInstance(
  name: string,
  config: ScopedLspServerConfig,
): LSPServerInstance {
  // ========== 验证未实现字段 ==========
  if (config.restartOnCrash !== undefined) {
    throw new Error(`restartOnCrash is not yet implemented`)
  }
  if (config.shutdownTimeout !== undefined) {
    throw new Error(`shutdownTimeout is not yet implemented`)
  }
  
  // ========== 惰性加载LSPClient ==========
  // 使用require()而非import，避免vscode-jsonrpc被静态加载
  const { createLSPClient } = require('./LSPClient.js')
  
  // ========== 私有状态 ==========
  let state: LspServerState = 'stopped'
  let startTime: Date | undefined
  let lastError: Error | undefined
  let restartCount = 0
  let crashRecoveryCount = 0
  
  // ========== 创建底层客户端 ==========
  const client = createLSPClient(name, error => {
    // 崩溃回调：设置状态为error
    state = 'error'
    lastError = error
    crashRecoveryCount++
  })
  
  // 返回公共接口...
}
```

**惰性加载设计：**
- `require()`在函数调用时执行
- 避免`vscode-jsonrpc`(~129KB)在静态导入链中加载
- 只有实际使用LSP功能时才加载

### 3.4 start() 方法详解

```typescript
async function start(): Promise<void> {
  // ========== 1. 状态检查 ==========
  if (state === 'running' || state === 'starting') return
  
  // ========== 2. 崩溃恢复限制 ==========
  const maxRestarts = config.maxRestarts ?? 3
  if (state === 'error' && crashRecoveryCount > maxRestarts) {
    const error = new Error(`exceeded max crash recovery attempts (${maxRestarts})`)
    lastError = error
    throw error
  }
  
  let initPromise: Promise<unknown> | undefined
  
  try {
    state = 'starting'
    
    // ========== 3. 启动客户端 ==========
    await client.start(config.command, config.args || [], {
      env: config.env,
      cwd: config.workspaceFolder,
    })
    
    // ========== 4. 构建初始化参数 ==========
    const workspaceFolder = config.workspaceFolder || getCwd()
    const workspaceUri = pathToFileURL(workspaceFolder).href
    
    const initParams: InitializeParams = {
      processId: process.pid,
      
      // 插件配置中的初始化选项
      initializationOptions: config.initializationOptions ?? {},
      
      // LSP 3.16+ 现代方式（Pyright, gopls需要）
      workspaceFolders: [
        { uri: workspaceUri, name: path.basename(workspaceFolder) }
      ],
      
      // 已弃用但某些服务器仍需要
      rootPath: workspaceFolder,   // typescript-language-server需要
      rootUri: workspaceUri,       // LSP 3.8弃用但仍有用
      
      // ========== 客户端能力声明 ==========
      capabilities: {
        workspace: {
          configuration: false,        // 不支持workspace/configuration
          workspaceFolders: false,     // 不支持文件夹变更通知
        },
        textDocument: {
          synchronization: {
            dynamicRegistration: false,
            willSave: false,
            willSaveWaitUntil: false,
            didSave: true,
          },
          publishDiagnostics: {
            relatedInformation: true,
            tagSupport: { valueSet: [1, 2] },  // Unnecessary, Deprecated
            versionSupport: false,
            codeDescriptionSupport: true,
            dataSupport: false,
          },
          hover: {
            contentFormat: ['markdown', 'plaintext'],
          },
          definition: { linkSupport: true },
          references: {},
          documentSymbol: {
            hierarchicalDocumentSymbolSupport: true,
          },
          callHierarchy: {},
        },
        general: {
          positionEncodings: ['utf-16'],
        },
      },
    }
    
    // ========== 5. 发送initialize请求 ==========
    initPromise = client.initialize(initParams)
    
    // ========== 6. 超时控制 ==========
    if (config.startupTimeout !== undefined) {
      await withTimeout(
        initPromise,
        config.startupTimeout,
        `LSP server '${name}' timed out after ${config.startupTimeout}ms`
      )
    } else {
      await initPromise
    }
    
    // ========== 7. 成功启动 ==========
    state = 'running'
    startTime = new Date()
    crashRecoveryCount = 0
    
  } catch (error) {
    // 清理
    client.stop().catch(() => {})
    initPromise?.catch(() => {})
    state = 'error'
    lastError = error as Error
    throw error
  }
}
```

**初始化参数详解：**

| 字段 | 用途 | 备注 |
|------|------|------|
| `workspaceFolders` | 现代工作区指定 | LSP 3.16+，Pyright/gopls需要 |
| `rootPath/rootUri` | 已弃用工作区指定 | TypeScript服务器仍需要 |
| `initializationOptions` | 服务器特定配置 | vue-language-server需要 |
| `capabilities` | 客户端能力声明 | 声明支持的功能 |

### 3.5 sendRequest() 方法（含重试机制）

```typescript
async function sendRequest<T>(method: string, params: unknown): Promise<T> {
  // 健康检查
  if (!isHealthy()) {
    throw new Error(`Cannot send request: server is ${state}`)
  }
  
  let lastAttemptError: Error | undefined
  
  // ========== 重试循环 ==========
  for (let attempt = 0; attempt <= MAX_RETRIES_FOR_TRANSIENT_ERRORS; attempt++) {
    try {
      return await client.sendRequest(method, params)
    } catch (error) {
      lastAttemptError = error as Error
      
      // ========== 检查是否为"内容已修改"错误 ==========
      const errorCode = (error as { code?: number }).code
      const isContentModifiedError =
        typeof errorCode === 'number' && errorCode === LSP_ERROR_CONTENT_MODIFIED
      
      if (isContentModifiedError && attempt < MAX_RETRIES_FOR_TRANSIENT_ERRORS) {
        // 指数退避重试
        const delay = RETRY_BASE_DELAY_MS * Math.pow(2, attempt)
        logForDebugging(
          `LSP request got ContentModified error, retrying in ${delay}ms`
        )
        await sleep(delay)
        continue
      }
      
      // 非重试错误或达到最大重试次数
      break
    }
  }
  
  throw new Error(`LSP request '${method}' failed: ${lastAttemptError?.message}`)
}
```

**重试机制设计：**

```typescript
// 常量定义
const LSP_ERROR_CONTENT_MODIFIED = -32801
const MAX_RETRIES_FOR_TRANSIENT_ERRORS = 3
const RETRY_BASE_DELAY_MS = 500
```

**典型场景：**
- rust-analyzer在项目索引期间返回ContentModified错误
- 这是预期行为，客户端应静默重试
- 重试延迟：500ms → 1000ms → 2000ms

### 3.6 isHealthy() 方法

```typescript
function isHealthy(): boolean {
  return state === 'running' && client.isInitialized
}
```

**双重检查：**
- 状态为`running`
- 客户端已完成初始化（收到initialize响应并发送initialized）

### 3.7 stop() 方法

```typescript
async function stop(): Promise<void> {
  if (state === 'stopped' || state === 'stopping') return
  
  try {
    state = 'stopping'
    await client.stop()
    state = 'stopped'
  } catch (error) {
    state = 'error'
    lastError = error as Error
    throw error
  }
}
```

### 3.8 restart() 方法

```typescript
async function restart(): Promise<void> {
  // 1. 停止
  try {
    await stop()
  } catch (error) {
    throw new Error(`Failed to stop during restart: ${errorMessage(error)}`)
  }
  
  // 2. 增加重启计数
  restartCount++
  
  // 3. 检查重启限制
  const maxRestarts = config.maxRestarts ?? 3
  if (restartCount > maxRestarts) {
    throw new Error(`Max restart attempts (${maxRestarts}) exceeded`)
  }
  
  // 4. 启动
  try {
    await start()
  } catch (error) {
    throw new Error(`Failed to start during restart (attempt ${restartCount}/${maxRestarts})`)
  }
}
```

---

## 4. LSPServerManager.ts 分析

**文件路径:** `services/lsp/LSPServerManager.ts`  
**核心职责:** 多服务器管理，文件路由，生命周期协调

### 4.1 类型定义

```typescript
export type LSPServerManager = {
  initialize(): Promise<void>                    // 加载配置并初始化
  shutdown(): Promise<void>                      // 关闭所有服务器
  
  getServerForFile(filePath): LSPServerInstance | undefined
  ensureServerStarted(filePath): Promise<LSPServerInstance | undefined>
  sendRequest<T>(filePath, method, params): Promise<T | undefined>
  
  getAllServers(): Map<string, LSPServerInstance>
  
  // 文件同步方法
  openFile(filePath, content): Promise<void>
  changeFile(filePath, content): Promise<void>
  saveFile(filePath): Promise<void>
  closeFile(filePath): Promise<void>
  isFileOpen(filePath): boolean
}
```

### 4.2 createLSPServerManager 工厂函数

```typescript
export function createLSPServerManager(): LSPServerManager {
  // ========== 私有状态 ==========
  const servers: Map<string, LSPServerInstance> = new Map()
  const extensionMap: Map<string, string[]> = new Map()  // 扩展名 → 服务器名列表
  const openedFiles: Map<string, string> = new Map()     // URI → 服务器名
  
  // 公共方法...
}
```

### 4.3 initialize() 方法

```typescript
async function initialize(): Promise<void> {
  let serverConfigs: Record<string, ScopedLspServerConfig>
  
  try {
    // ========== 1. 加载所有LSP服务器配置 ==========
    const result = await getAllLspServers()
    serverConfigs = result.servers
  } catch (error) {
    throw error
  }
  
  // ========== 2. 构建扩展名映射 ==========
  for (const [serverName, config] of Object.entries(serverConfigs)) {
    try {
      // 验证必需字段
      if (!config.command) {
        throw new Error(`Server ${serverName} missing required 'command' field`)
      }
      if (!config.extensionToLanguage || Object.keys(config.extensionToLanguage).length === 0) {
        throw new Error(`Server ${serverName} missing 'extensionToLanguage' field`)
      }
      
      // 映射文件扩展名
      const fileExtensions = Object.keys(config.extensionToLanguage)
      for (const ext of fileExtensions) {
        const normalized = ext.toLowerCase()
        if (!extensionMap.has(normalized)) {
          extensionMap.set(normalized, [])
        }
        extensionMap.get(normalized)?.push(serverName)
      }
      
      // ========== 3. 创建服务器实例 ==========
      const instance = createLSPServerInstance(serverName, config)
      servers.set(serverName, instance)
      
      // ========== 4. 注册反向请求处理器 ==========
      // 处理workspace/configuration请求（某些服务器会发送）
      instance.onRequest(
        'workspace/configuration',
        (params: { items: Array<{ section?: string }> }) => {
          // 返回null/空配置
          return params.items.map(() => null)
        }
      )
      
    } catch (error) {
      // 继续处理其他服务器，不失败整体初始化
      logError(new Error(`Failed to initialize ${serverName}: ${error.message}`))
    }
  }
}
```

**extensionMap结构：**
```
extensionMap: {
  '.ts'   => ['typescript-language-server'],
  '.js'   => ['typescript-language-server'],
  '.vue'  => ['vue-language-server'],
  '.py'   => ['pyright'],
  '.go'   => ['gopls'],
}
```

### 4.4 getServerForFile() 方法

```typescript
function getServerForFile(filePath: string): LSPServerInstance | undefined {
  const ext = path.extname(filePath).toLowerCase()
  const serverNames = extensionMap.get(ext)
  
  if (!serverNames || serverNames.length === 0) return undefined
  
  // 使用第一个注册的服务器
  const serverName = serverNames[0]
  return servers.get(serverName)
}
```

### 4.5 ensureServerStarted() 方法

```typescript
async function ensureServerStarted(filePath: string): Promise<LSPServerInstance | undefined> {
  const server = getServerForFile(filePath)
  if (!server) return undefined
  
  // 如果服务器停止或出错，启动它
  if (server.state === 'stopped' || server.state === 'error') {
    try {
      await server.start()
    } catch (error) {
      throw error
    }
  }
  
  return server
}
```

### 4.6 文件同步方法

```typescript
async function openFile(filePath: string, content: string): Promise<void> {
  const server = await ensureServerStarted(filePath)
  if (!server) return
  
  const fileUri = pathToFileURL(path.resolve(filePath)).href
  
  // ========== 检查是否已打开 ==========
  if (openedFiles.get(fileUri) === server.name) {
    logForDebugging(`File already open, skipping didOpen for ${filePath}`)
    return
  }
  
  // ========== 获取语言ID ==========
  const ext = path.extname(filePath).toLowerCase()
  const languageId = server.config.extensionToLanguage[ext] || 'plaintext'
  
  try {
    // ========== 发送didOpen通知 ==========
    await server.sendNotification('textDocument/didOpen', {
      textDocument: {
        uri: fileUri,
        languageId,
        version: 1,
        text: content,
      },
    })
    
    // 记录打开状态
    openedFiles.set(fileUri, server.name)
    
  } catch (error) {
    throw new Error(`Failed to sync file open ${filePath}: ${errorMessage(error)}`)
  }
}

async function changeFile(filePath: string, content: string): Promise<void> {
  const server = getServerForFile(filePath)
  if (!server || server.state !== 'running') {
    // 如果服务器未运行，降级为openFile
    return openFile(filePath, content)
  }
  
  const fileUri = pathToFileURL(path.resolve(filePath)).href
  
  // 如果文件未打开，先打开
  if (openedFiles.get(fileUri) !== server.name) {
    return openFile(filePath, content)
  }
  
  try {
    await server.sendNotification('textDocument/didChange', {
      textDocument: { uri: fileUri, version: 1 },
      contentChanges: [{ text: content }],
    })
  } catch (error) {
    throw new Error(`Failed to sync file change ${filePath}: ${errorMessage(error)}`)
  }
}

async function saveFile(filePath: string): Promise<void> {
  const server = getServerForFile(filePath)
  if (!server || server.state !== 'running') return
  
  try {
    await server.sendNotification('textDocument/didSave', {
      textDocument: { uri: pathToFileURL(path.resolve(filePath)).href },
    })
  } catch (error) {
    throw new Error(`Failed to sync file save ${filePath}: ${errorMessage(error)}`)
  }
}

async function closeFile(filePath: string): Promise<void> {
  const server = getServerForFile(filePath)
  if (!server || server.state !== 'running') return
  
  const fileUri = pathToFileURL(path.resolve(filePath)).href
  
  try {
    await server.sendNotification('textDocument/didClose', {
      textDocument: { uri: fileUri },
    })
    
    // 移除打开记录
    openedFiles.delete(fileUri)
    
  } catch (error) {
    throw new Error(`Failed to sync file close ${filePath}: ${errorMessage(error)}`)
  }
}

function isFileOpen(filePath: string): boolean {
  const fileUri = pathToFileURL(path.resolve(filePath)).href
  return openedFiles.has(fileUri)
}
```

**文件同步流程：**
```
┌──────────────────────────────────────────────────────────┐
│                    文件同步生命周期                         │
├──────────────────────────────────────────────────────────┤
│                                                           │
│  用户编辑文件                                              │
│       │                                                   │
│       v                                                   │
│  ┌──────────┐                                             │
│  │ openFile │  didOpen通知                                │
│  └──────────┘                                             │
│       │                                                   │
│       v                                                   │
│  ┌──────────┐                                             │
│  │changeFile│  didChange通知（循环）                       │
│  └──────────┘                                             │
│       │                                                   │
│       v                                                   │
│  ┌──────────┐                                             │
│  │ saveFile │  didSave通知                                │
│  └──────────┘                                             │
│       │                                                   │
│       v                                                   │
│  ┌──────────┐                                             │
│  │closeFile │  didClose通知                               │
│  └──────────┘                                             │
│                                                           │
│  注意：didOpen必须在didChange之前                          │
│       openedFiles Map用于跟踪状态                          │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

### 4.7 shutdown() 方法

```typescript
async function shutdown(): Promise<void> {
  // ========== 只停止运行中或出错的服务器 ==========
  const toStop = Array.from(servers.entries()).filter(
    ([, s]) => s.state === 'running' || s.state === 'error'
  )
  
  // ========== 并行停止 ==========
  const results = await Promise.allSettled(
    toStop.map(([, server]) => server.stop())
  )
  
  // ========== 清理状态 ==========
  servers.clear()
  extensionMap.clear()
  openedFiles.clear()
  
  // ========== 收集错误 ==========
  const errors = results
    .map((r, i) =>
      r.status === 'rejected'
        ? `${toStop[i]![0]}: ${errorMessage(r.reason)}`
        : null
    )
    .filter((e): e is string => e !== null)
  
  if (errors.length > 0) {
    throw new Error(`Failed to stop ${errors.length} LSP server(s): ${errors.join('; ')}`)
  }
}
```

---

## 5. 语言服务器配置

### 5.1 配置Schema定义

**文件路径:** `utils/plugins/schemas.ts`

```typescript
export const LspServerConfigSchema = lazySchema(() =>
  z.strictObject({
    // ========== 必需字段 ==========
    command: z.string().min(1)
      .refine(cmd => {
        // 命令不应包含空格（使用args数组）
        if (cmd.includes(' ') && !cmd.startsWith('/')) return false
        return true
      }, { message: 'Command should not contain spaces. Use args array.' })
      .describe('Command to execute the LSP server'),
    
    extensionToLanguage: z.record(fileExtension(), nonEmptyString())
      .refine(record => Object.keys(record).length > 0, {
        message: 'extensionToLanguage must have at least one mapping'
      })
      .describe('Mapping from file extension to LSP language ID'),
    
    // ========== 可选字段 ==========
    args: z.array(nonEmptyString()).optional()
      .describe('Command-line arguments'),
    
    transport: z.enum(['stdio', 'socket']).default('stdio')
      .describe('Communication transport mechanism'),
    
    env: z.record(z.string(), z.string()).optional()
      .describe('Environment variables'),
    
    initializationOptions: z.unknown().optional()
      .describe('Initialization options passed to the server'),
    
    settings: z.unknown().optional()
      .describe('Settings via workspace/didChangeConfiguration'),
    
    workspaceFolder: z.string().optional()
      .describe('Workspace folder path'),
    
    startupTimeout: z.number().int().positive().optional()
      .describe('Maximum time to wait for server startup (ms)'),
    
    shutdownTimeout: z.number().int().positive().optional()
      .describe('Maximum time to wait for graceful shutdown (ms)'),
    
    restartOnCrash: z.boolean().optional()
      .describe('Whether to restart the server if it crashes'),
    
    maxRestarts: z.number().int().nonnegative().optional()
      .describe('Maximum number of restart attempts'),
  })
)
```

### 5.2 配置示例

#### TypeScript Language Server

```json
{
  "typescript-language-server": {
    "command": "typescript-language-server",
    "args": ["--stdio"],
    "extensionToLanguage": {
      ".ts": "typescript",
      ".tsx": "typescriptreact",
      ".js": "javascript",
      ".jsx": "javascriptreact"
    },
    "startupTimeout": 10000,
    "maxRestarts": 3
  }
}
```

#### Python (Pyright)

```json
{
  "pyright": {
    "command": "pyright",
    "args": ["--stdio"],
    "extensionToLanguage": {
      ".py": "python",
      ".pyi": "python"
    },
    "initializationOptions": {
      "analysis": {
        "autoSearchPaths": true,
        "diagnosticMode": "openFilesOnly"
      }
    }
  }
}
```

#### Go (gopls)

```json
{
  "gopls": {
    "command": "gopls",
    "args": ["mode=stdio"],
    "extensionToLanguage": {
      ".go": "go"
    },
    "env": {
      "GOFLAGS": "-mod=mod"
    }
  }
}
```

#### Vue Language Server

```json
{
  "vue-language-server": {
    "command": "vue-language-server",
    "args": ["--stdio"],
    "extensionToLanguage": {
      ".vue": "vue"
    },
    "initializationOptions": {
      "typescript": {
        "serverPath": "/path/to/typescript-language-server"
      }
    }
  }
}
```

### 5.3 插件配置加载流程

**文件路径:** `services/lsp/config.ts`

```typescript
export async function getAllLspServers(): Promise<{
  servers: Record<string, ScopedLspServerConfig>
}> {
  const allServers: Record<string, ScopedLspServerConfig> = {}
  
  try {
    // ========== 1. 加载所有启用的插件 ==========
    const { enabled: plugins } = await loadAllPluginsCacheOnly()
    
    // ========== 2. 并行加载各插件的LSP配置 ==========
    const results = await Promise.all(
      plugins.map(async plugin => {
        const errors: PluginError[] = []
        try {
          const scopedServers = await getPluginLspServers(plugin, errors)
          return { plugin, scopedServers, errors }
        } catch (e) {
          // 单插件失败不影响其他插件
          return { plugin, scopedServers: undefined, errors }
        }
      })
    )
    
    // ========== 3. 合并所有服务器配置 ==========
    for (const { plugin, scopedServers, errors } of results) {
      if (scopedServers && Object.keys(scopedServers).length > 0) {
        Object.assign(allServers, scopedServers)
      }
      
      if (errors.length > 0) {
        logForDebugging(`${errors.length} error(s) loading from plugin: ${plugin.name}`)
      }
    }
    
  } catch (error) {
    // LSP是可选功能，失败不抛出
    logError(toError(error))
  }
  
  return { servers: allServers }
}
```

### 5.4 插件作用域机制

**文件路径:** `utils/plugins/lspPluginIntegration.ts`

```typescript
export function addPluginScopeToLspServers(
  servers: Record<string, LspServerConfig>,
  pluginName: string,
): Record<string, ScopedLspServerConfig> {
  const scopedServers: Record<string, ScopedLspServerConfig> = {}
  
  for (const [name, config] of Object.entries(servers)) {
    // ========== 添加插件前缀避免冲突 ==========
    const scopedName = `plugin:${pluginName}:${name}`
    scopedServers[scopedName] = {
      ...config,
      scope: 'dynamic',
      source: pluginName,
    }
  }
  
  return scopedServers
}
```

**作用域命名格式：**
```
plugin:<pluginName>:<serverName>

示例：
plugin:typescript-plugin:typescript-language-server
plugin:python-plugin:pyright
```

### 5.5 环境变量解析

```typescript
export function resolvePluginLspEnvironment(
  config: LspServerConfig,
  plugin: { path: string; source: string },
  userConfig?: PluginOptionValues,
): LspServerConfig {
  const resolveValue = (value: string): string => {
    // 1. 替换插件特定变量
    let resolved = substitutePluginVariables(value, plugin)
    
    // 2. 替换用户配置变量
    if (userConfig) {
      resolved = substituteUserConfigVariables(resolved, userConfig)
    }
    
    // 3. 扩展通用环境变量
    const { expanded, missingVars } = expandEnvVarsInString(resolved)
    
    return expanded
  }
  
  const resolved = { ...config }
  
  // 解析命令路径
  if (resolved.command) {
    resolved.command = resolveValue(resolved.command)
  }
  
  // 解析参数
  if (resolved.args) {
    resolved.args = resolved.args.map(arg => resolveValue(arg))
  }
  
  // 添加插件特定环境变量
  const resolvedEnv: Record<string, string> = {
    CLAUDE_PLUGIN_ROOT: plugin.path,
    CLAUDE_PLUGIN_DATA: getPluginDataDir(plugin.source),
    ...(resolved.env || {}),
  }
  
  resolved.env = resolvedEnv
  
  return resolved
}
```

**支持的变量：**
| 变量 | 说明 |
|------|------|
| `${CLAUDE_PLUGIN_ROOT}` | 插件目录路径 |
| `${CLAUDE_PLUGIN_DATA}` | 插件数据目录 |
| `${user_config.KEY}` | 用户配置值 |
| `${ENV_VAR}` | 系统环境变量 |

---

## 6. 诊断信息处理

### 6.1 架构概览

```
┌────────────────────────────────────────────────────────────┐
│                    诊断信息处理流程                           │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  LSP Server                                                 │
│      │                                                      │
│      │ publishDiagnostics notification                      │
│      v                                                      │
│  ┌─────────────────────────────────────────────┐            │
│  │ passiveFeedback.ts                          │            │
│  │ registerLSPNotificationHandlers()           │            │
│  │  - 接收诊断通知                              │            │
│  │  - 格式化为Claude格式                        │            │
│  │  - 注册到pending队列                         │            │
│  └───────────────────────┬─────────────────────┘            │
│                           │                                 │
│                           v                                 │
│  ┌─────────────────────────────────────────────┐            │
│  │ LSPDiagnosticRegistry.ts                    │            │
│  │ registerPendingLSPDiagnostic()              │            │
│  │  - 存储待交付诊断                            │            │
│  │ checkForLSPDiagnostics()                    │            │
│  │  - 去重、限容、返回待交付                    │            │
│  └───────────────────────┬─────────────────────┘            │
│                           │                                 │
│                           v                                 │
│  ┌─────────────────────────────────────────────┐            │
│  │ Claude Attachment System                    │            │
│  │  - 自动附加到对话                           │            │
│  │  - 用户可见诊断信息                         │            │
│  └─────────────────────────────────────────────┘            │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

### 6.2 passiveFeedback.ts 分析

**文件路径:** `services/lsp/passiveFeedback.ts`

#### 诊断格式转换

```typescript
function mapLSPSeverity(lspSeverity: number | undefined): 'Error' | 'Warning' | 'Info' | 'Hint' {
  // LSP DiagnosticSeverity enum:
  // 1 = Error, 2 = Warning, 3 = Information, 4 = Hint
  switch (lspSeverity) {
    case 1: return 'Error'
    case 2: return 'Warning'
    case 3: return 'Info'
    case 4: return 'Hint'
    default: return 'Error'
  }
}

export function formatDiagnosticsForAttachment(
  params: PublishDiagnosticsParams,
): DiagnosticFile[] {
  // ========== 解析URI ==========
  let uri: string
  try {
    uri = params.uri.startsWith('file://')
      ? fileURLToPath(params.uri)  // 转换file://URI为路径
      : params.uri                 // 使用原始路径
  } catch (error) {
    // 容错：使用原始URI
    uri = params.uri
  }
  
  // ========== 转换诊断格式 ==========
  const diagnostics = params.diagnostics.map(diag => ({
    message: diag.message,
    severity: mapLSPSeverity(diag.severity),
    range: {
      start: {
        line: diag.range.start.line,
        character: diag.range.start.character,
      },
      end: {
        line: diag.range.end.line,
        character: diag.range.end.character,
      },
    },
    source: diag.source,
    code: diag.code !== undefined && diag.code !== null
      ? String(diag.code)
      : undefined,
  }))
  
  return [{ uri, diagnostics }]
}
```

#### 注册诊断处理器

```typescript
export function registerLSPNotificationHandlers(
  manager: LSPServerManager,
): HandlerRegistrationResult {
  const servers = manager.getAllServers()
  
  const registrationErrors: Array<{ serverName: string; error: string }> = []
  const diagnosticFailures: Map<string, { count: number; lastError: string }> = new Map()
  let successCount = 0
  
  // ========== 为每个服务器注册处理器 ==========
  for (const [serverName, serverInstance] of servers.entries()) {
    try {
      serverInstance.onNotification(
        'textDocument/publishDiagnostics',
        (params: unknown) => {
          try {
            // ========== 验证参数结构 ==========
            if (!params || typeof params !== 'object' ||
                !('uri' in params) || !('diagnostics' in params)) {
              logError(new Error(`Invalid diagnostic params from ${serverName}`))
              return
            }
            
            const diagnosticParams = params as PublishDiagnosticsParams
            
            // ========== 格式化诊断 ==========
            const diagnosticFiles = formatDiagnosticsForAttachment(diagnosticParams)
            
            // ========== 检查是否有诊断 ==========
            const firstFile = diagnosticFiles[0]
            if (!firstFile || firstFile.diagnostics.length === 0) {
              return
            }
            
            // ========== 注册待交付诊断 ==========
            registerPendingLSPDiagnostic({
              serverName,
              files: diagnosticFiles,
            })
            
            // 成功：重置失败计数
            diagnosticFailures.delete(serverName)
            
          } catch (error) {
            // ========== 失败处理 ==========
            const failures = diagnosticFailures.get(serverName) || { count: 0, lastError: '' }
            failures.count++
            failures.lastError = errorMessage(error)
            diagnosticFailures.set(serverName, failures)
            
            // ========== 3次以上失败警告 ==========
            if (failures.count >= 3) {
              logForDebugging(
                `WARNING: LSP diagnostic handler for ${serverName} has failed ${failures.count} times`
              )
            }
          }
        }
      )
      
      successCount++
      
    } catch (error) {
      registrationErrors.push({ serverName, error: errorMessage(error) })
    }
  }
  
  return { totalServers: servers.size, successCount, registrationErrors, diagnosticFailures }
}
```

### 6.3 LSPDiagnosticRegistry.ts 分析

**文件路径:** `services/lsp/LSPDiagnosticRegistry.ts`

#### 数据结构

```typescript
export type PendingLSPDiagnostic = {
  serverName: string           // 服务器名称
  files: DiagnosticFile[]      // 诊断文件列表
  timestamp: number            // 接收时间
  attachmentSent: boolean      // 是否已发送
}

// 诊断文件格式
type DiagnosticFile = {
  uri: string                  // 文件URI
  diagnostics: Array<{
    message: string            // 诊断消息
    severity: 'Error' | 'Warning' | 'Info' | 'Hint'
    range: {                   // 位置范围
      start: { line: number; character: number }
      end: { line: number; character: number }
    }
    source?: string            // 来源（如'typescript'）
    code?: string              // 错误代码
  }>
}
```

#### 全局状态

```typescript
// 容量限制常量
const MAX_DIAGNOSTICS_PER_FILE = 10   // 每文件最多10条诊断
const MAX_TOTAL_DIAGNOSTICS = 30      // 总共最多30条诊断
const MAX_DELIVERED_FILES = 500       // LRU缓存最多500个文件

// 全局注册表
const pendingDiagnostics = new Map<string, PendingLSPDiagnostic>()

// 跨轮次去重缓存（LRU）
const deliveredDiagnostics = new LRUCache<string, Set<string>>({
  max: MAX_DELIVERED_FILES,
})
```

#### 注册待交付诊断

```typescript
export function registerPendingLSPDiagnostic({
  serverName,
  files,
}: {
  serverName: string
  files: DiagnosticFile[]
}): void {
  // 使用UUID保证唯一性
  const diagnosticId = randomUUID()
  
  pendingDiagnostics.set(diagnosticId, {
    serverName,
    files,
    timestamp: Date.now(),
    attachmentSent: false,
  })
}
```

#### 去重机制

```typescript
function createDiagnosticKey(diag: {
  message: string
  severity?: string
  range?: unknown
  source?: string
  code?: unknown
}): string {
  // 使用JSON序列化作为唯一键
  return jsonStringify({
    message: diag.message,
    severity: diag.severity,
    range: diag.range,
    source: diag.source || null,
    code: diag.code || null,
  })
}

function deduplicateDiagnosticFiles(allFiles: DiagnosticFile[]): DiagnosticFile[] {
  const fileMap = new Map<string, Set<string>>()
  const dedupedFiles: DiagnosticFile[] = []
  
  for (const file of allFiles) {
    if (!fileMap.has(file.uri)) {
      fileMap.set(file.uri, new Set())
      dedupedFiles.push({ uri: file.uri, diagnostics: [] })
    }
    
    const seenDiagnostics = fileMap.get(file.uri)!
    const dedupedFile = dedupedFiles.find(f => f.uri === file.uri)!
    
    // 获取之前交付的诊断（跨轮次去重）
    const previouslyDelivered = deliveredDiagnostics.get(file.uri) || new Set()
    
    for (const diag of file.diagnostics) {
      const key = createDiagnosticKey(diag)
      
      // 跳过已在当前批次或之前轮次中出现的诊断
      if (seenDiagnostics.has(key) || previouslyDelivered.has(key)) {
        continue
      }
      
      seenDiagnostics.add(key)
      dedupedFile.diagnostics.push(diag)
    }
  }
  
  return dedupedFiles.filter(f => f.diagnostics.length > 0)
}
```

#### 获取待交付诊断

```typescript
export function checkForLSPDiagnostics(): Array<{
  serverName: string
  files: DiagnosticFile[]
}> {
  // ========== 1. 收集所有未发送诊断 ==========
  const allFiles: DiagnosticFile[] = []
  const serverNames = new Set<string>()
  const diagnosticsToMark: PendingLSPDiagnostic[] = []
  
  for (const diagnostic of pendingDiagnostics.values()) {
    if (!diagnostic.attachmentSent) {
      allFiles.push(...diagnostic.files)
      serverNames.add(diagnostic.serverName)
      diagnosticsToMark.push(diagnostic)
    }
  }
  
  if (allFiles.length === 0) return []
  
  // ========== 2. 去重 ==========
  let dedupedFiles = deduplicateDiagnosticFiles(allFiles)
  
  // ========== 3. 标记已发送并删除 ==========
  for (const diagnostic of diagnosticsToMark) {
    diagnostic.attachmentSent = true
  }
  for (const [id, diagnostic] of pendingDiagnostics) {
    if (diagnostic.attachmentSent) {
      pendingDiagnostics.delete(id)
    }
  }
  
  // ========== 4. 容量限制 ==========
  let totalDiagnostics = 0
  let truncatedCount = 0
  
  for (const file of dedupedFiles) {
    // 按严重性排序（Error优先）
    file.diagnostics.sort(
      (a, b) => severityToNumber(a.severity) - severityToNumber(b.severity)
    )
    
    // 每文件限制
    if (file.diagnostics.length > MAX_DIAGNOSTICS_PER_FILE) {
      truncatedCount += file.diagnostics.length - MAX_DIAGNOSTICS_PER_FILE
      file.diagnostics = file.diagnostics.slice(0, MAX_DIAGNOSTICS_PER_FILE)
    }
    
    // 总量限制
    const remainingCapacity = MAX_TOTAL_DIAGNOSTICS - totalDiagnostics
    if (file.diagnostics.length > remainingCapacity) {
      truncatedCount += file.diagnostics.length - remainingCapacity
      file.diagnostics = file.diagnostics.slice(0, remainingCapacity)
    }
    
    totalDiagnostics += file.diagnostics.length
  }
  
  dedupedFiles = dedupedFiles.filter(f => f.diagnostics.length > 0)
  
  // ========== 5. 记录已交付诊断（跨轮次去重） ==========
  for (const file of dedupedFiles) {
    if (!deliveredDiagnostics.has(file.uri)) {
      deliveredDiagnostics.set(file.uri, new Set())
    }
    const delivered = deliveredDiagnostics.get(file.uri)!
    for (const diag of file.diagnostics) {
      delivered.add(createDiagnosticKey(diag))
    }
  }
  
  // ========== 6. 返回结果 ==========
  if (dedupedFiles.length === 0) return []
  
  return [
    {
      serverName: Array.from(serverNames).join(', '),
      files: dedupedFiles,
    }
  ]
}
```

#### 清理函数

```typescript
// 清除所有待处理诊断
export function clearAllLSPDiagnostics(): void {
  pendingDiagnostics.clear()
}

// 重置所有状态（包括跨轮次跟踪）
export function resetAllLSPDiagnosticState(): void {
  pendingDiagnostics.clear()
  deliveredDiagnostics.clear()
}

// 清除特定文件的已交付诊断
export function clearDeliveredDiagnosticsForFile(fileUri: string): void {
  deliveredDiagnostics.delete(fileUri)
}

// 获取待处理数量
export function getPendingLSPDiagnosticCount(): number {
  return pendingDiagnostics.size
}
```

---

## 7. 用户使用指南

### 7.1 manager.ts 全局管理

**文件路径:** `services/lsp/manager.ts`

#### 初始化流程

```typescript
// 全局单例
let lspManagerInstance: LSPServerManager | undefined
let initializationState: InitializationState = 'not-started'
let initializationError: Error | undefined
let initializationGeneration = 0
let initializationPromise: Promise<void> | undefined

export function initializeLspServerManager(): void {
  // ========== 1. 检查bare模式 ==========
  if (isBareMode()) return  // --bare模式不启用LSP
  
  // ========== 2. 检查是否已初始化 ==========
  if (lspManagerInstance !== undefined && initializationState !== 'failed') {
    return
  }
  
  // ========== 3. 创建管理器实例 ==========
  lspManagerInstance = createLSPServerManager()
  initializationState = 'pending'
  
  // ========== 4. 开始异步初始化 ==========
  const currentGeneration = ++initializationGeneration
  initializationPromise = lspManagerInstance
    .initialize()
    .then(() => {
      if (currentGeneration === initializationGeneration) {
        initializationState = 'success'
        
        // 注册诊断处理器
        if (lspManagerInstance) {
          registerLSPNotificationHandlers(lspManagerInstance)
        }
      }
    })
    .catch((error: unknown) => {
      if (currentGeneration === initializationGeneration) {
        initializationState = 'failed'
        initializationError = error as Error
        lspManagerInstance = undefined
      }
    })
}
```

#### API访问

```typescript
// 获取管理器实例
export function getLspServerManager(): LSPServerManager | undefined {
  if (initializationState === 'failed') return undefined
  return lspManagerInstance
}

// 获取初始化状态
export function getInitializationStatus():
  | { status: 'not-started' }
  | { status: 'pending' }
  | { status: 'success' }
  | { status: 'failed'; error: Error }

// 检查是否有服务器连接
export function isLspConnected(): boolean {
  if (initializationState === 'failed') return false
  const manager = getLspServerManager()
  if (!manager) return false
  const servers = manager.getAllServers()
  if (servers.size === 0) return false
  for (const server of servers.values()) {
    if (server.state !== 'error') return true
  }
  return false
}
```

#### 关闭流程

```typescript
export async function shutdownLspServerManager(): Promise<void> {
  if (lspManagerInstance === undefined) return
  
  try {
    await lspManagerInstance.shutdown()
  } catch (error) {
    logError(error as Error)
  } finally {
    // 清理状态
    lspManagerInstance = undefined
    initializationState = 'not-started'
    initializationError = undefined
    initializationPromise = undefined
    initializationGeneration++
  }
}
```

### 7.2 使用示例

#### 基本使用流程

```typescript
import { 
  initializeLspServerManager,
  getLspServerManager,
  shutdownLspServerManager
} from './services/lsp/manager.js'

// ========== 1. 启动时初始化 ==========
initializeLspServerManager()

// ========== 2. 使用LSP功能 ==========
const manager = getLspServerManager()
if (manager) {
  // 打开文件
  await manager.openFile('/path/to/file.ts', fileContent)
  
  // 发送请求（如跳转定义）
  const result = await manager.sendRequest(
    '/path/to/file.ts',
    'textDocument/definition',
    {
      textDocument: { uri: 'file:///path/to/file.ts' },
      position: { line: 10, character: 5 }
    }
  )
  
  // 文件变更
  await manager.changeFile('/path/to/file.ts', newContent)
  
  // 关闭文件
  await manager.closeFile('/path/to/file.ts')
}

// ========== 3. 关闭时清理 ==========
await shutdownLspServerManager()
```

#### 获取诊断信息

```typescript
import { checkForLSPDiagnostics } from './services/lsp/LSPDiagnosticRegistry.js'

// 定期检查待交付诊断
const diagnostics = checkForLSPDiagnostics()

if (diagnostics.length > 0) {
  for (const { serverName, files } of diagnostics) {
    console.log(`Diagnostics from ${serverName}:`)
    for (const file of files) {
      console.log(`  File: ${file.uri}`)
      for (const diag of file.diagnostics) {
        console.log(`    [${diag.severity}] Line ${diag.range.start.line}: ${diag.message}`)
      }
    }
  }
}
```

### 7.3 插件开发指南

#### 创建LSP插件

1. **创建插件目录结构：**

```
my-lsp-plugin/
├── manifest.json
├── .lsp.json
└── README.md
```

2. **manifest.json配置：**

```json
{
  "name": "my-lsp-plugin",
  "version": "1.0.0",
  "description": "LSP support for MyLanguage",
  "lspServers": ".lsp.json"
}
```

3. **.lsp.json配置：**

```json
{
  "my-language-server": {
    "command": "${CLAUDE_PLUGIN_ROOT}/bin/my-language-server",
    "args": ["--stdio"],
    "extensionToLanguage": {
      ".myl": "mylanguage"
    },
    "initializationOptions": {
      "featureX": true
    }
  }
}
```

#### 使用用户配置

```json
// manifest.json
{
  "name": "my-lsp-plugin",
  "userConfig": {
    "serverPath": {
      "type": "string",
      "description": "Path to language server executable",
      "default": ""
    }
  }
}

// .lsp.json
{
  "my-language-server": {
    "command": "${user_config.serverPath}",
    "args": ["--stdio"],
    "extensionToLanguage": { ".myl": "mylanguage" }
  }
}
```

---

## 8. 从零实现

### 8.1 最小化LSP客户端实现

```typescript
// minimal-lsp-client.ts
import { spawn, type ChildProcess } from 'child_process'
import { createMessageConnection, StreamMessageReader, StreamMessageWriter } from 'vscode-jsonrpc/node.js'

export class MinimalLSPClient {
  private process: ChildProcess | null = null
  private connection: any = null
  
  async start(command: string, args: string[] = []): Promise<void> {
    // 1. 启动进程
    this.process = spawn(command, args, {
      stdio: ['pipe', 'pipe', 'pipe']
    })
    
    // 2. 等待进程启动
    await new Promise<void>((resolve, reject) => {
      this.process!.once('spawn', resolve)
      this.process!.once('error', reject)
    })
    
    // 3. 创建连接
    const reader = new StreamMessageReader(this.process.stdout)
    const writer = new StreamMessageWriter(this.process.stdin)
    this.connection = createMessageConnection(reader, writer)
    
    // 4. 开始监听
    this.connection.listen()
  }
  
  async initialize(rootPath: string): Promise<void> {
    // 发送initialize请求
    const result = await this.connection.sendRequest('initialize', {
      processId: process.pid,
      rootUri: `file://${rootPath}`,
      capabilities: {}
    })
    
    // 发送initialized通知
    await this.connection.sendNotification('initialized', {})
  }
  
  async openFile(uri: string, languageId: string, content: string): Promise<void> {
    await this.connection.sendNotification('textDocument/didOpen', {
      textDocument: { uri, languageId, version: 1, text: content }
    })
  }
  
  async getDiagnostics(uri: string): Promise<any[]> {
    // 注意：诊断通常通过publishDiagnostics通知接收
    // 这里只是一个示例
    return []
  }
  
  async stop(): Promise<void> {
    if (this.connection) {
      await this.connection.sendRequest('shutdown', {})
      await this.connection.sendNotification('exit', {})
      this.connection.dispose()
    }
    
    if (this.process) {
      this.process.kill()
    }
  }
  
  // 注册诊断处理器
  onDiagnostics(handler: (params: any) => void): void {
    this.connection.onNotification('textDocument/publishDiagnostics', handler)
  }
}
```

### 8.2 使用示例

```typescript
// usage-example.ts
import { MinimalLSPClient } from './minimal-lsp-client.js'

async function main() {
  const client = new MinimalLSPClient()
  
  try {
    // 启动服务器
    await client.start('typescript-language-server', ['--stdio'])
    
    // 初始化
    await client.initialize('/path/to/project')
    
    // 注册诊断处理器
    client.onDiagnostics((params) => {
      console.log('Diagnostics received:', params)
    })
    
    // 打开文件
    const fileContent = `const x: string = 123;  // 类型错误`
    await client.openFile('file:///path/to/file.ts', 'typescript', fileContent)
    
    // 等待诊断...
    await new Promise(r => setTimeout(r, 2000))
    
  } finally {
    await client.stop()
  }
}

main().catch(console.error)
```

### 8.3 完整实现步骤

#### 步骤1：底层通信

```typescript
// 1. 安装依赖
// npm install vscode-jsonrpc vscode-languageserver-protocol

// 2. 创建进程
const process = spawn(command, args, { stdio: ['pipe', 'pipe', 'pipe'] })

// 3. 创建JSON-RPC连接
const reader = new StreamMessageReader(process.stdout)
const writer = new StreamMessageWriter(process.stdin)
const connection = createMessageConnection(reader, writer)

// 4. 监听消息
connection.listen()
```

#### 步骤2：初始化流程

```typescript
// 1. 发送initialize请求
const initResult = await connection.sendRequest('initialize', {
  processId: process.pid,
  rootUri: workspaceUri,
  workspaceFolders: [{ uri: workspaceUri, name: workspaceName }],
  capabilities: {
    textDocument: {
      synchronization: { didSave: true },
      publishDiagnostics: { relatedInformation: true }
    }
  }
})

// 2. 保存服务器能力
const capabilities = initResult.capabilities

// 3. 发送initialized通知
await connection.sendNotification('initialized', {})
```

#### 步骤3：文件同步

```typescript
// 1. 打开文件
await connection.sendNotification('textDocument/didOpen', {
  textDocument: {
    uri: fileUri,
    languageId: languageId,
    version: 1,
    text: content
  }
})

// 2. 变更文件
await connection.sendNotification('textDocument/didChange', {
  textDocument: { uri: fileUri, version: 2 },
  contentChanges: [{ text: newContent }]
})

// 3. 保存文件
await connection.sendNotification('textDocument/didSave', {
  textDocument: { uri: fileUri }
})

// 4. 关闭文件
await connection.sendNotification('textDocument/didClose', {
  textDocument: { uri: fileUri }
})
```

#### 步骤4：接收诊断

```typescript
// 注册诊断处理器
connection.onNotification('textDocument/publishDiagnostics', (params) => {
  const { uri, diagnostics } = params
  
  for (const diag of diagnostics) {
    console.log(`[${diag.severity}] ${diag.message}`)
    console.log(`  at line ${diag.range.start.line}`)
  }
})
```

#### 步骤5：请求功能

```typescript
// 跳转定义
const definition = await connection.sendRequest('textDocument/definition', {
  textDocument: { uri: fileUri },
  position: { line: 10, character: 5 }
})

// 悬停信息
const hover = await connection.sendRequest('textDocument/hover', {
  textDocument: { uri: fileUri },
  position: { line: 10, character: 5 }
})

// 查找引用
const references = await connection.sendRequest('textDocument/references', {
  textDocument: { uri: fileUri },
  position: { line: 10, character: 5 },
  context: { includeDeclaration: true }
})
```

#### 步骤6：关闭流程

```typescript
// 1. 发送shutdown请求
await connection.sendRequest('shutdown', {})

// 2. 发送exit通知
await connection.sendNotification('exit', {})

// 3. 清理连接
connection.dispose()

// 4. 终止进程
process.kill()
```

### 8.4 关键设计考虑

| 考虑点 | 解决方案 | 代码位置 |
|--------|----------|----------|
| 进程启动异步性 | 等待'spawn'事件 | `LSPClient.ts:116-131` |
| 连接错误处理 | 先注册错误处理器再listen | `LSPClient.ts:185-207` |
| 区分主动关闭与崩溃 | `isStopping`标志 | `LSPClient.ts:62` |
| 惰性加载大依赖 | `require()`而非`import` | `LSPServerInstance.ts:110` |
| 崩溃恢复限制 | `crashRecoveryCount`计数 | `LSPServerInstance.ts:142-149` |
| ContentModified重试 | 指数退避重试 | `LSPServerInstance.ts:367-401` |
| 文件同步状态跟踪 | `openedFiles`Map | `LSPServerManager.ts:64` |
| 诊断去重 | JSON序列化键 + LRU缓存 | `LSPDiagnosticRegistry.ts:136-184` |
| 诊断容量限制 | 按严重性排序截断 | `LSPDiagnosticRegistry.ts:256-288` |
| 插件作用域 | 前缀命名避免冲突 | `lspPluginIntegration.ts:298-315` |

---

## 附录

### A. 文件依赖关系图

```
┌─────────────────────────────────────────────────────────────┐
│                    services/lsp/ 依赖图                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  manager.ts (全局管理)                                       │
│      ├── LSPServerManager.ts                                │
│      │      ├── LSPServerInstance.ts                        │
│      │      │      └── LSPClient.ts                         │
│      │      │              └── vscode-jsonrpc               │
│      │      ├── config.ts                                   │
│      │      │      └── lspPluginIntegration.ts              │
│      │      │              └── schemas.ts                   │
│      │      └── passiveFeedback.ts                          │
│      │              └── LSPDiagnosticRegistry.ts            │
│      └── passiveFeedback.ts                                 │
│                                                              │
│  外部依赖:                                                   │
│      ├── vscode-jsonrpc (JSON-RPC通信)                      │
│      ├── vscode-languageserver-protocol (LSP类型)           │
│      ├── lru-cache (诊断去重缓存)                            │
│      └── child_process (进程管理)                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### B. LSP错误码参考

| 错误码 | 名称 | 说明 |
|--------|------|------|
| -32700 | ParseError | JSON解析错误 |
| -32600 | InvalidRequest | 无效请求 |
| -32601 | MethodNotFound | 方法不存在 |
| -32602 | InvalidParams | 无效参数 |
| -32603 | InternalError | 内部错误 |
| -32801 | ContentModified | 内容已修改（需重试） |
| -32802 | RequestCancelled | 请求取消 |

### C. 常用LSP语言服务器

| 语言 | 服务器 | 安装命令 |
|------|--------|----------|
| TypeScript | typescript-language-server | `npm install -g typescript-language-server` |
| JavaScript | typescript-language-server | 同上 |
| Python | pyright | `npm install -g pyright` |
| Go | gopls | Go扩展自动安装 |
| Rust | rust-analyzer | rustup组件 |
| Vue | vue-language-server | `npm install -g @vue/language-server` |
| Java | jdtls | Eclipse JDT Language Server |
| C/C++ | clangd | LLVM工具链 |

---

*文档生成日期: 2026-04-02*  
*分析源码版本: Claude Code CLI*