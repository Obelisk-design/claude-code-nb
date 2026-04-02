# Chrome扩展集成详解

**分析日期:** 2026-04-02

## 目录

1. [Chrome集成概述](#1-chrome集成概述)
2. [commands/chrome/ - 命令模块](#2-commandschrome---命令模块)
3. [utils/claudeInChrome/ - 核心工具模块](#3-utilsclaudeinchrome---核心工具模块)
4. [Native Messaging机制](#4-native-messaging机制)
5. [MCP服务器实现](#5-mcp服务器实现)
6. [用户使用指南](#6-用户使用指南)
7. [从零实现Chrome扩展集成](#7-从零实现chrome扩展集成)

---

## 1. Chrome集成概述

### 1.1 架构全景

Claude Code的Chrome扩展集成是一个完整的浏览器自动化解决方案，允许Claude直接控制用户的Chrome浏览器进行网页操作。

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Claude Code CLI                                   │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────┐   │
│  │ chrome命令   │    │ Skill系统    │    │ MCP Client           │   │
│  │ (UI菜单)     │    │ (激活入口)   │    │ (工具调用)           │   │
│  └──────────────┘    └──────────────┘    └──────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    MCP Server (子进程)                               │
│  --claude-in-chrome-mcp 启动                                         │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ mcpServer.ts → createClaudeForChromeMcpServer()               │   │
│  │ 提供 BROWSER_TOOLS (17个浏览器工具)                           │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼ WebSocket/Unix Socket
┌─────────────────────────────────────────────────────────────────────┐
│                    Native Messaging Host                             │
│  --chrome-native-host 启动                                           │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ chromeNativeHost.ts → ChromeNativeHost类                      │   │
│  │ 管理Socket服务器，转发消息                                     │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼ Native Messaging协议
┌─────────────────────────────────────────────────────────────────────┐
│                    Chrome Extension                                  │
│  claude-for-chrome (安装在浏览器中)                                  │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ 接收工具请求 → 执行浏览器操作 → 返回结果                      │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 核心文件清单

| 文件路径 | 功能 |
|---------|------|
| `commands/chrome/index.ts` | 命令注册入口 |
| `commands/chrome/chrome.tsx` | Chrome设置菜单UI组件 |
| `utils/claudeInChrome/common.ts` | 浏览器检测、配置常量、Socket路径管理 |
| `utils/claudeInChrome/setup.ts` | MCP配置生成、Native Host安装 |
| `utils/claudeInChrome/setupPortable.ts` | 可移植的扩展检测（VSCode/TUI共用） |
| `utils/claudeInChrome/mcpServer.ts` | MCP服务器主进程实现 |
| `utils/claudeInChrome/chromeNativeHost.ts` | Native Messaging Host实现 |
| `utils/claudeInChrome/prompt.ts` | Chrome工具系统提示词 |
| `utils/claudeInChrome/toolRendering.tsx` | 工具调用UI渲染定制 |
| `skills/bundled/claudeInChrome.ts` | 内置Skill注册 |
| `hooks/useChromeExtensionNotification.tsx` | 启动时扩展检测通知 |
| `hooks/usePromptsFromClaudeInChrome.tsx` | 从扩展接收提示请求 |
| `entrypoints/cli.tsx` | CLI快速路径入口 |

### 1.3 支持的浏览器

```typescript
// common.ts 第39-216行
export const CHROMIUM_BROWSERS: Record<ChromiumBrowser, BrowserConfig> = {
  chrome: { name: 'Google Chrome', ... },
  brave: { name: 'Brave', ... },
  arc: { name: 'Arc', ... },
  edge: { name: 'Microsoft Edge', ... },
  chromium: { name: 'Chromium', ... },
  vivaldi: { name: 'Vivaldi', ... },
  opera: { name: 'Opera', ... },
}
```

支持7种Chromium内核浏览器，覆盖macOS、Linux、Windows三大平台。

---

## 2. commands/chrome/ - 命令模块

### 2.1 index.ts - 命令注册

```typescript
// commands/chrome/index.ts
import { getIsNonInteractiveSession } from '../../bootstrap/state.js'
import type { Command } from '../../commands.js'

const command: Command = {
  name: 'chrome',
  description: 'Claude in Chrome (Beta) settings',
  availability: ['claude-ai'],  // 仅claude.ai用户可用
  isEnabled: () => !getIsNonInteractiveSession(),  // 非交互会话禁用
  type: 'local-jsx',
  load: () => import('./chrome.js'),
}

export default command
```

**关键点：**
- `availability: ['claude-ai']` - 只有claude.ai订阅用户可使用
- 非交互会话（SDK、CI模式）自动禁用
- 使用`local-jsx`类型，动态加载React组件

### 2.2 chrome.tsx - 菜单UI组件

**文件位置:** `commands/chrome/chrome.tsx`

**组件结构：**

```tsx
// 第17-24行 Props定义
type MenuAction = 'install-extension' | 'reconnect' | 'manage-permissions' | 'toggle-default'

type Props = {
  onDone: (result?: string) => void
  isExtensionInstalled: boolean
  configEnabled: boolean | undefined
  isClaudeAISubscriber: boolean
  isWSL: boolean
}
```

**菜单选项（第74-110行 handleAction）：**

| Action | 功能 |
|--------|------|
| `install-extension` | 打开 https://claude.ai/chrome 安装页面 |
| `reconnect` | 重新检测扩展并打开重连页面 |
| `manage-permissions` | 打开权限管理页面 |
| `toggle-default` | 切换默认启用状态 |

**渲染逻辑（第189-254行）：**

```tsx
const isDisabled = isWSL || true && !isClaudeAISubscriber

// 显示状态信息
<Text>Status: {isConnected ? 'Enabled' : 'Disabled'}</Text>
<Text>Extension: {isExtensionInstalled ? 'Installed' : 'Not detected'}</Text>

// 使用提示
<Text>Usage: claude --chrome or claude --no-chrome</Text>
```

**call函数入口（第278-284行）：**

```tsx
export const call = async function (onDone) {
  const isExtensionInstalled = await isChromeExtensionInstalled()
  const config = getGlobalConfig()
  const isSubscriber = isClaudeAISubscriber()
  const isWSL = env.isWslEnvironment()
  return <ClaudeInChromeMenu ... />
}
```

---

## 3. utils/claudeInChrome/ - 核心工具模块

### 3.1 common.ts - 通用配置

**扩展ID定义：**

```typescript
// common.ts 第6-12行
export const CHROME_EXTENSION_URL = 'https://claude.ai/chrome'

const PROD_EXTENSION_ID = 'fcoeoabgfenejglbffodgkkbkcdhcgfn'
const DEV_EXTENSION_ID = 'dihbgbndebgnbjfmelmegjepbnkhlgni'
const ANT_EXTENSION_ID = 'dngcpimnedloihjnnfngkgjoidhnaolf'

function getExtensionIds(): string[] {
  return process.env.USER_TYPE === 'ant'
    ? [PROD_EXTENSION_ID, DEV_EXTENSION_ID, ANT_EXTENSION_ID]
    : [PROD_EXTENSION_ID]
}
```

**浏览器检测顺序（第38-46行）：**

```typescript
const BROWSER_DETECTION_ORDER: ChromiumBrowser[] = [
  'chrome', 'brave', 'arc', 'edge', 'chromium', 'vivaldi', 'opera',
]
```

**Native Messaging Hosts路径（第280-317行）：**

```typescript
export function getAllNativeMessagingHostsDirs() {
  // macOS: ~/Library/Application Support/[Browser]/NativeMessagingHosts
  // Linux: ~/.config/[Browser]/NativeMessagingHosts  
  // Windows: 使用注册表而非文件路径
}
```

**Socket路径管理（第481-527行）：**

```typescript
// Unix Socket路径
export function getSecureSocketPath(): string {
  if (platform() === 'win32') {
    return `\\\\.\\pipe\\${getSocketName()}`  // Windows命名管道
  }
  return join(getSocketDir(), `${process.pid}.sock`)  // Unix socket
}

export function getAllSocketPaths(): string[] {
  // 扫描/tmp/claude-mcp-browser-bridge-[user]/目录下的所有.sock文件
  // 包含遗留路径作为fallback
}

function getSocketDir(): string {
  return `/tmp/claude-mcp-browser-bridge-${getUsername()}`
}
```

**openInChrome函数（第429-469行）：**

```typescript
export async function openInChrome(url: string): Promise<boolean> {
  const browser = await detectAvailableBrowser()
  
  switch (currentPlatform) {
    case 'macos':
      await execFileNoThrow('open', ['-a', config.macos.appName, url])
    case 'windows':
      await execFileNoThrow('rundll32', ['url,OpenURL', url])
    case 'linux':
      await execFileNoThrow(binary, [url])
  }
}
```

### 3.2 setup.ts - 设置安装逻辑

**启用判断逻辑（第39-68行）：**

```typescript
export function shouldEnableClaudeInChrome(chromeFlag?: boolean): boolean {
  // 1. 非交互会话默认禁用
  if (getIsNonInteractiveSession() && chromeFlag !== true) return false
  
  // 2. CLI标志优先
  if (chromeFlag === true) return true
  if (chromeFlag === false) return false
  
  // 3. 环境变量
  if (isEnvTruthy(process.env.CLAUDE_CODE_ENABLE_CFC)) return true
  if (isEnvDefinedFalsy(process.env.CLAUDE_CODE_ENABLE_CFC)) return false
  
  // 4. 全局配置
  const config = getGlobalConfig()
  if (config.claudeInChromeDefaultEnabled !== undefined) {
    return config.claudeInChromeDefaultEnabled
  }
  
  return false
}
```

**自动启用逻辑（第70-84行）：**

```typescript
export function shouldAutoEnableClaudeInChrome(): boolean {
  shouldAutoEnable =
    getIsInteractive() &&
    isChromeExtensionInstalled_CACHED_MAY_BE_STALE() &&
    (process.env.USER_TYPE === 'ant' ||
      getFeatureValue_CACHED_MAY_BE_STALE('tengu_chrome_auto_enable', false))
  return shouldAutoEnable
}
```

**setupClaudeInChrome函数（第91-171行）：**

```typescript
export function setupClaudeInChrome(): {
  mcpConfig: Record<string, ScopedMcpServerConfig>
  allowedTools: string[]
  systemPrompt: string
} {
  const allowedTools = BROWSER_TOOLS.map(
    tool => `mcp__claude-in-chrome__${tool.name}`
  )

  // 创建Native Host wrapper脚本
  void createWrapperScript(execCommand)
    .then(manifestBinaryPath => installChromeNativeHostManifest(manifestBinaryPath))
  
  // 返回MCP配置
  return {
    mcpConfig: {
      'claude-in-chrome': {
        type: 'stdio',
        command: process.execPath,
        args: ['--claude-in-chrome-mcp'],
        scope: 'dynamic',
      }
    },
    allowedTools,
    systemPrompt: getChromeSystemPrompt(),
  }
}
```

**Native Host Manifest安装（第191-266行）：**

```typescript
export async function installChromeNativeHostManifest(manifestBinaryPath) {
  const manifest = {
    name: 'com.anthropic.claude_code_browser_extension',
    description: 'Claude Code Browser Extension Native Host',
    path: manifestBinaryPath,
    type: 'stdio',
    allowed_origins: [
      'chrome-extension://fcoeoabgfenejglbffodgkkbkcdhcgfn/',
      // ant用户额外包含DEV和ANT扩展ID
    ],
  }
  
  // 安装到所有浏览器的NativeMessagingHosts目录
  for (const manifestDir of manifestDirs) {
    await mkdir(manifestDir, { recursive: true })
    await writeFile(manifestPath, jsonStringify(manifest, null, 2))
  }
  
  // Windows需要注册表项
  if (getPlatform() === 'windows') {
    registerWindowsNativeHosts(manifestPath)
  }
}
```

**Wrapper脚本创建（第308-346行）：**

```typescript
async function createWrapperScript(command: string): Promise<string> {
  const wrapperPath = platform === 'windows'
    ? join(chromeDir, 'chrome-native-host.bat')
    : join(chromeDir, 'chrome-native-host')
  
  const scriptContent = platform === 'windows'
    ? `@echo off\nREM Chrome native host wrapper script\n${command}`
    : `#!/bin/sh\nexec ${command}`
  
  await writeFile(wrapperPath, scriptContent)
  await chmod(wrapperPath, 0o755)  // Unix需要执行权限
  return wrapperPath
}
```

### 3.3 setupPortable.ts - 可移植扩展检测

**用途：** 同时被TUI和VSCode扩展使用，无平台依赖的纯检测逻辑。

```typescript
// setupPortable.ts 第97-135行
export function getAllBrowserDataPathsPortable(): BrowserPath[] {
  const home = homedir()
  const paths: BrowserPath[] = []
  
  for (const browserId of BROWSER_DETECTION_ORDER) {
    const config = CHROMIUM_BROWSERS[browserId]
    
    switch (process.platform) {
      case 'darwin': dataPath = config.macos
      case 'linux': dataPath = config.linux
      case 'win32': {
        const appDataBase = config.windows.useRoaming
          ? join(home, 'AppData', 'Roaming')
          : join(home, 'AppData', 'Local')
        paths.push({ browser: browserId, path: join(appDataBase, ...) })
      }
    }
  }
  return paths
}
```

**扩展检测核心逻辑（第147-213行）：**

```typescript
export async function detectExtensionInstallationPortable(browserPaths, log?) {
  for (const { browser, path: browserBasePath } of browserPaths) {
    // 读取浏览器profile目录
    const profileDirs = entries
      .filter(entry => entry.name === 'Default' || entry.name.startsWith('Profile '))
    
    // 检查每个profile中是否存在扩展ID目录
    for (const profile of profileDirs) {
      for (const extensionId of extensionIds) {
        const extensionPath = join(browserBasePath, profile, 'Extensions', extensionId)
        await readdir(extensionPath)  // 成功则扩展存在
        return { isInstalled: true, browser }
      }
    }
  }
  return { isInstalled: false, browser: null }
}
```

### 3.4 prompt.ts - 系统提示词

```typescript
// prompt.ts 第1-46行
export const BASE_CHROME_PROMPT = `
# Claude in Chrome browser automation

You have access to browser automation tools (mcp__claude-in-chrome__*) ...

## GIF recording
When performing multi-step browser interactions, use mcp__claude-in-chrome__gif_creator ...

## Console log debugging
Use mcp__claude-in-chrome__read_console_messages with 'pattern' parameter ...

## Alerts and dialogs
IMPORTANT: Do not trigger JavaScript alerts, confirms, prompts... 
These browser dialogs block all further browser events...

## Avoid rabbit holes and loops
When using browser automation tools, stay focused...
- Browser tool calls failing after 2-3 attempts
- No response from the browser extension
...

## Tab context and session startup
IMPORTANT: At the start of each browser automation session, 
call mcp__claude-in-chrome__tabs_context_mcp first...
`
```

**Tool Search指令（第52-61行）：**

```typescript
export const CHROME_TOOL_SEARCH_INSTRUCTIONS = `
**IMPORTANT: Before using any chrome browser tools, 
you MUST first load them using ToolSearch.**

Before calling any mcp__claude-in-chrome__* tool:
1. Use ToolSearch with select:mcp__claude-in-chrome__<tool_name>
2. Then call the tool
`
```

**Skill Hint（第76-84行）：**

```typescript
export const CLAUDE_IN_CHROME_SKILL_HINT = `
**Browser Automation**: Chrome browser tools are available 
via the "claude-in-chrome" skill. CRITICAL: Before using 
any mcp__claude-in-chrome__* tools, invoke the skill by 
calling the Skill tool with skill: "claude-in-chrome".
`
```

---

## 4. Native Messaging机制

### 4.1 概述

Chrome Native Messaging是Chrome扩展与本机应用程序通信的标准协议。

**通信流程：**

```
Chrome Extension ←→ Native Host (CLI子进程) ←→ MCP Server ←→ Claude Code
     │                    STDIO                       Socket
     │                (Native Messaging)              (MCP协议)
     │
     └── chrome.runtime.sendNativeMessage()
```

### 4.2 chromeNativeHost.ts详解

**消息格式（第49-57行）：**

```typescript
// Native Messaging协议：4字节长度前缀 + JSON消息
export function sendChromeMessage(message: string): void {
  const jsonBytes = Buffer.from(message, 'utf-8')
  const lengthBuffer = Buffer.alloc(4)
  lengthBuffer.writeUInt32LE(jsonBytes.length, 0)  // 小端序32位长度
  
  process.stdout.write(lengthBuffer)  // 先写长度
  process.stdout.write(jsonBytes)     // 再写内容
}
```

**主入口（第59-82行）：**

```typescript
export async function runChromeNativeHost(): Promise<void> {
  const host = new ChromeNativeHost()
  const messageReader = new ChromeMessageReader()
  
  await host.start()  // 启动Socket服务器
  
  // 循环读取Chrome消息
  while (true) {
    const message = await messageReader.read()
    if (message === null) break  // stdin关闭，Chrome断开
    
    await host.handleMessage(message)
  }
  
  await host.stop()
}
```

**ChromeNativeHost类（第103-434行）：**

```typescript
class ChromeNativeHost {
  private mcpClients = new Map<number, McpClient>()
  private server: Server | null = null
  private socketPath: string | null = null
  
  async start(): Promise<void> {
    this.socketPath = getSecureSocketPath()
    
    // Unix: 创建安全目录并清理陈旧socket
    if (platform() !== 'win32') {
      await mkdir(socketDir, { recursive: true, mode: 0o700 })
      // 清理已死进程的socket文件
      for (const file of files) {
        const pid = parseInt(file.replace('.sock', ''), 10)
        try { process.kill(pid, 0) } catch { await unlink(file) }
      }
    }
    
    // 创建Socket服务器
    this.server = createServer(socket => this.handleMcpClient(socket))
    await this.server.listen(this.socketPath)
  }
  
  async handleMessage(messageJson: string): Promise<void> {
    const message = messageSchema().parse(jsonParse(messageJson))
    
    switch (message.type) {
      case 'ping':  // 心跳检测
        sendChromeMessage(jsonStringify({ type: 'pong', timestamp: Date.now() }))
        
      case 'get_status':  // 状态查询
        sendChromeMessage(jsonStringify({ type: 'status_response', native_host_version: VERSION }))
        
      case 'tool_response':  // 工具响应，转发给MCP客户端
        for (const [id, client] of this.mcpClients) {
          client.socket.write(responseMsg)
        }
        
      case 'notification':  // 通知消息
        for (const [id, client] of this.mcpClients) {
          client.socket.write(notificationMsg)
        }
    }
  }
  
  private handleMcpClient(socket: Socket): void {
    // MCP客户端连接时通知Chrome
    sendChromeMessage(jsonStringify({ type: 'mcp_connected' }))
    
    socket.on('data', (data: Buffer) => {
      // 解析MCP协议消息（同样是长度前缀格式）
      const request = jsonParse(messageBytes.toString('utf-8'))
      
      // 转发给Chrome扩展
      sendChromeMessage(jsonStringify({
        type: 'tool_request',
        method: request.method,
        params: request.params,
      }))
    })
  }
}
```

**ChromeMessageReader类（第440-527行）：**

```typescript
class ChromeMessageReader {
  private buffer = Buffer.alloc(0)
  private pendingResolve: ((value: string | null) => void) | null = null
  
  constructor() {
    process.stdin.on('data', (chunk: Buffer) => {
      this.buffer = Buffer.concat([this.buffer, chunk])
      this.tryProcessMessage()
    })
    
    process.stdin.on('end', () => {
      this.closed = true
      if (this.pendingResolve) this.pendingResolve(null)
    })
  }
  
  async read(): Promise<string | null> {
    // 检查buffer中是否有完整消息
    if (this.buffer.length >= 4) {
      const length = this.buffer.readUInt32LE(0)
      if (length > 0 && length <= MAX_MESSAGE_SIZE && this.buffer.length >= 4 + length) {
        const messageBytes = this.buffer.subarray(4, 4 + length)
        this.buffer = this.buffer.subarray(4 + length)
        return messageBytes.toString('utf-8')
      }
    }
    
    // 等待更多数据
    return new Promise(resolve => { this.pendingResolve = resolve })
  }
}
```

### 4.3 消息协议总结

**Chrome → Native Host:**
```json
{
  "type": "tool_response",
  "id": 123,
  "result": { ... }
}
```

**Native Host → MCP Client:**
```
[4字节长度][JSON: {"id": 123, "result": {...}}]
```

**MCP Client → Native Host:**
```json
{
  "method": "tools/call",
  "params": { "name": "navigate", "arguments": { "url": "..." } }
}
```

---

## 5. MCP服务器实现

### 5.1 mcpServer.ts详解

**创建Chrome Context（第85-246行）：**

```typescript
export function createChromeContext(env?: Record<string, string>): ClaudeForChromeContext {
  const logger = new DebugLogger()
  const chromeBridgeUrl = getChromeBridgeUrl()  // WebSocket Bridge URL
  
  return {
    serverName: 'Claude in Chrome',
    logger,
    socketPath: getSecureSocketPath(),
    getSocketPaths: getAllSocketPaths,
    clientTypeId: 'claude-code',
    
    // 认证错误回调
    onAuthenticationError: () => {
      logger.warn('Authentication error. Please ensure you are logged into...')
    },
    
    // 断开连接回调
    onToolCallDisconnected: () => {
      return 'Browser extension is not connected. Please ensure...'
    },
    
    // 设备配对回调（保存到配置）
    onExtensionPaired: (deviceId: string, name: string) => {
      saveGlobalConfig(config => ({
        ...config,
        chromeExtension: { pairedDeviceId: deviceId, pairedDeviceName: name },
      }))
    },
    
    // 获取持久化的设备ID
    getPersistedDeviceId: () => getGlobalConfig().chromeExtension?.pairedDeviceId,
    
    // Bridge配置（WebSocket连接）
    bridgeConfig: {
      url: chromeBridgeUrl,
      getUserId: async () => getGlobalConfig().oauthAccount?.accountUuid,
      getOAuthToken: async () => getClaudeAIOAuthTokens()?.accessToken ?? '',
    },
    
    // Ant专属：browser_task工具的Anthropic API调用
    callAnthropicMessages: async (req) => {
      const response = await sideQuery({
        model: req.model,
        system: req.system,
        messages: req.messages,
        skipSystemPromptPrefix: true,
        tools: [],  // 关键：阻止Sonnet生成XML function_calls
      })
      return { content: textBlocks, stop_reason: response.stop_reason, usage }
    },
    
    // 事件追踪
    trackEvent: (eventName, metadata) => {
      // 只转发安全的元数据键，排除可能包含用户数据的字段
      logEvent(eventName, safeMetadata)
    },
  }
}
```

**Bridge URL解析（第51-72行）：**

```typescript
function getChromeBridgeUrl(): string | undefined {
  const bridgeEnabled =
    process.env.USER_TYPE === 'ant' ||
    getFeatureValue_CACHED_MAY_BE_STALE('tengu_copper_bridge', false)
  
  if (!bridgeEnabled) return undefined  // 使用Native Socket
  
  if (isEnvTruthy(process.env.LOCAL_BRIDGE)) {
    return 'ws://localhost:8765'  // 本地开发
  }
  if (isEnvTruthy(process.env.USE_STAGING_OAUTH)) {
    return 'wss://bridge-staging.claudeusercontent.com'
  }
  
  return 'wss://bridge.claudeusercontent.com'  // 生产环境
}
```

**运行MCP服务器（第248-275行）：**

```typescript
export async function runClaudeInChromeMcpServer(): Promise<void> {
  enableConfigs()
  initializeAnalyticsSink()
  const context = createChromeContext()
  
  const server = createClaudeForChromeMcpServer(context)
  const transport = new StdioServerTransport()
  
  // 父进程退出时优雅关闭
  const shutdownAndExit = async () => {
    await shutdown1PEventLogging()
    await shutdownDatadog()
    process.exit(0)
  }
  process.stdin.on('end', () => void shutdownAndExit())
  process.stdin.on('error', () => void shutdownAndExit())
  
  await server.connect(transport)
}
```

### 5.2 CLI快速路径

```typescript
// entrypoints/cli.tsx 第72-85行
if (process.argv[2] === '--claude-in-chrome-mcp') {
  profileCheckpoint('cli_claude_in_chrome_mcp_path')
  const { runClaudeInChromeMcpServer } = await import('../utils/claudeInChrome/mcpServer.js')
  await runClaudeInChromeMcpServer()
  return
} else if (process.argv[2] === '--chrome-native-host') {
  profileCheckpoint('cli_chrome_native_host_path')
  const { runChromeNativeHost } = await import('../utils/claudeInChrome/chromeNativeHost.js')
  await runChromeNativeHost()
  return
}
```

### 5.3 提供的工具列表

来自 `@ant/claude-for-chrome-mcp` 包的 `BROWSER_TOOLS`：

| 工具名 | 功能 |
|--------|------|
| `javascript_tool` | 执行JavaScript代码 |
| `read_page` | 读取页面内容 |
| `find` | 查找页面元素 |
| `form_input` | 表单输入 |
| `computer` | 计算机操作（点击、输入等） |
| `navigate` | 页面导航 |
| `resize_window` | 窗口大小调整 |
| `gif_creator` | GIF录制 |
| `upload_image` | 图片上传 |
| `get_page_text` | 获取页面文本 |
| `tabs_context_mcp` | 获取标签页上下文 |
| `tabs_create_mcp` | 创建新标签页 |
| `update_plan` | 更新执行计划 |
| `read_console_messages` | 读取控制台日志 |
| `read_network_requests` | 读取网络请求 |
| `shortcuts_list` | 快捷方式列表 |
| `shortcuts_execute` | 执行快捷方式 |

---

## 6. 用户使用指南

### 6.1 安装步骤

1. **安装Chrome扩展：**
   - 访问 https://claude.ai/chrome
   - 在Chrome浏览器中安装Claude for Chrome扩展
   - 登录claude.ai账号（需与Claude Code使用的账号一致）

2. **首次连接：**
   - 启动Claude Code时会自动检测扩展
   - 如果检测到扩展，会自动打开reconnect页面完成配对
   - 或手动运行 `/chrome` 命令选择 "Reconnect extension"

3. **权限设置：**
   - 在Chrome扩展设置中配置站点权限
   - 控制Claude可以浏览、点击、输入的网站

### 6.2 使用方式

**命令行标志：**

```bash
# 启用Chrome集成
claude --chrome

# 禁用Chrome集成
claude --no-chrome

# 默认行为由配置决定
claude
```

**Skill激活：**

```
# 用户直接调用
/skill claude-in-chrome

# 或在对话中提及浏览器操作
"帮我打开这个网页并截图"
```

**菜单管理：**

```
/chrome
# 显示菜单：
# - Install Chrome extension
# - Manage permissions
# - Reconnect extension  
# - Enabled by default: Yes/No
```

### 6.3 启动通知

**检测逻辑（hooks/useChromeExtensionNotification.tsx）：**

```typescript
// 需要订阅但未订阅
if (!isClaudeAISubscriber()) {
  return { jsx: <Text color="error">Claude in Chrome requires a subscription</Text> }
}

// 扩展未安装
if (!installed) {
  return { jsx: <Text color="warning">Chrome extension not detected</Text> }
}

// 默认启用时显示提示
if (chromeFlag === undefined) {
  return { text: "Claude in Chrome enabled · /chrome" }
}
```

### 6.4 从扩展接收提示

**hooks/usePromptsFromClaudeInChrome.tsx：**

```typescript
// 监听扩展发送的prompt通知
mcpClient.client.setNotificationHandler(
  ClaudeInChromePromptNotificationSchema(),
  notification => {
    const { tabId, prompt, image } = notification.params
    
    // 只处理我们追踪的tab（避免处理随机通知）
    if (!isTrackedClaudeInChromeTabId(tabId)) return
    
    // 将prompt添加到用户输入队列
    if (image) {
      // 包含图片截图
      enqueuePendingNotification({
        value: [{ type: 'text', text: prompt }, { type: 'image', source: image }],
        mode: 'prompt',
      })
    } else {
      enqueuePendingNotification({ value: prompt, mode: 'prompt' })
    }
  }
)

// 同步权限模式
useEffect(() => {
  const chromeMode = toolPermissionMode === 'bypassPermissions' 
    ? 'skip_all_permission_checks' 
    : 'ask'
  callIdeRpc('set_permission_mode', { mode: chromeMode }, chromeClient)
}, [mcpClients, toolPermissionMode])
```

---

## 7. 从零实现Chrome扩展集成

### 7.1 设计思路

**核心组件：**

1. **Native Messaging Host** - 与Chrome扩展通信的桥梁
2. **MCP Server** - 将浏览器能力暴露为工具
3. **设置/检测逻辑** - 扩展发现和配置管理
4. **UI组件** - 用户交互界面

### 7.2 Native Messaging Host实现要点

```typescript
// 关键代码模板
class NativeHost {
  async start() {
    // 1. 创建Socket服务器等待MCP客户端连接
    this.server = createServer(socket => this.handleClient(socket))
    
    // 2. 监听stdin接收Chrome消息
    process.stdin.on('data', chunk => {
      // Native Messaging协议：4字节长度前缀 + JSON
      const length = buffer.readUInt32LE(0)
      const message = buffer.slice(4, 4 + length).toString()
      this.handleChromeMessage(JSON.parse(message))
    })
  }
  
  handleChromeMessage(msg) {
    // 转发工具响应给MCP客户端
    if (msg.type === 'tool_response') {
      for (const client of this.mcpClients) {
        client.socket.write(encodeMcpMessage(msg))
      }
    }
  }
  
  handleClient(socket) {
    socket.on('data', data => {
      // 转发工具请求给Chrome
      const request = decodeMcpMessage(data)
      sendToChrome({ type: 'tool_request', ...request })
    })
  }
}

// Native Messaging发送函数
function sendToChrome(msg) {
  const json = JSON.stringify(msg)
  const lenBuf = Buffer.alloc(4)
  lenBuf.writeUInt32LE(json.length, 0)
  process.stdout.write(lenBuf)
  process.stdout.write(json)
}
```

### 7.3 MCP Server实现要点

```typescript
// 关键代码模板
export function setupChromeMcp() {
  return {
    mcpConfig: {
      'chrome-browser': {
        type: 'stdio',
        command: process.execPath,
        args: ['--chrome-mcp-server'],  // CLI快速路径
      }
    },
    allowedTools: [
      'mcp__chrome-browser__navigate',
      'mcp__chrome-browser__click',
      'mcp__chrome-browser__type',
      // ...
    ],
    systemPrompt: CHROME_SYSTEM_PROMPT,
  }
}

// MCP Server进程
async function runMcpServer() {
  const transport = new StdioServerTransport()
  const server = createMcpServer({
    name: 'chrome-browser',
    tools: BROWSER_TOOLS,  // 从外部包导入
    executeTool: async (name, args) => {
      // 通过Native Host发送给Chrome扩展执行
      return await sendToNativeHost({ tool: name, args })
    }
  })
  await server.connect(transport)
}
```

### 7.4 扩展检测实现要点

```typescript
// 检测扩展是否安装
async function detectExtension(extensionId: string) {
  const browserPaths = getBrowserDataPaths()
  
  for (const { browser, path } of browserPaths) {
    // 检查每个profile
    const profiles = ['Default', 'Profile 1', 'Profile 2', ...]
    
    for (const profile of profiles) {
      const extPath = join(path, profile, 'Extensions', extensionId)
      try {
        await readdir(extPath)
        return true  // 找到扩展
      } catch {
        continue
      }
    }
  }
  return false
}

// 获取浏览器数据路径
function getBrowserDataPaths() {
  const home = homedir()
  return [
    { browser: 'chrome', path: join(home, '.config/google-chrome') },
    { browser: 'brave', path: join(home, '.config/BraveSoftware/Brave-Browser') },
    // ...
  ]
}
```

### 7.5 Native Host Manifest安装

```typescript
async function installManifest() {
  const manifest = {
    name: 'com.mycompany.browser_extension',
    description: 'My Browser Extension Native Host',
    path: wrapperScriptPath,
    type: 'stdio',
    allowed_origins: ['chrome-extension://MY_EXTENSION_ID/'],
  }
  
  // macOS/Linux: 写入NativeMessagingHosts目录
  const manifestDir = join(home, '.config/google-chrome/NativeMessagingHosts')
  await writeFile(join(manifestDir, manifest.name + '.json'), JSON.stringify(manifest))
  
  // Windows: 注册表
  await execFile('reg', ['add', `HKCU\\Software\\Google\\Chrome\\NativeMessagingHosts\\${manifest.name}`, '/d', manifestPath])
}
```

### 7.6 UI组件模板

```tsx
function ChromeMenu({ onDone, isInstalled }) {
  const [enabled, setEnabled] = useState(false)
  
  const options = [
    { label: 'Install Extension', value: 'install' },
    { label: 'Reconnect', value: 'reconnect' },
    { label: `Enabled: ${enabled ? 'Yes' : 'No'}`, value: 'toggle' },
  ]
  
  return (
    <Dialog title="Chrome Integration">
      <Text>Status: {isInstalled ? 'Installed' : 'Not Detected'}</Text>
      <Select options={options} onChange={handleAction} />
      <Text dimColor>Usage: claude --chrome</Text>
    </Dialog>
  )
}
```

### 7.7 完整架构图

```
┌──────────────────────────────────────────────────────────────────┐
│                    实现清单                                       │
├──────────────────────────────────────────────────────────────────┤
│ 1. utils/browser/common.ts                                       │
│    - 浏览器配置常量                                               │
│    - 扩展ID定义                                                   │
│    - Socket路径管理                                               │
│                                                                   │
│ 2. utils/browser/setup.ts                                        │
│    - 扩展检测函数                                                 │
│    - Manifest安装                                                 │
│    - Wrapper脚本创建                                              │
│    - MCP配置生成                                                  │
│                                                                   │
│ 3. utils/browser/nativeHost.ts                                   │
│    - NativeHost类                                                 │
│    - Socket服务器                                                 │
│    - 消息转发                                                     │
│    - Chrome Messaging协议                                         │
│                                                                   │
│ 4. utils/browser/mcpServer.ts                                    │
│    - MCP Server入口                                               │
│    - Context配置                                                  │
│    - 工具执行                                                     │
│                                                                   │
│ 5. commands/browser/index.ts                                     │
│    - 命令注册                                                     │
│                                                                   │
│ 6. commands/browser/browser.tsx                                  │
│    - 菜单UI组件                                                   │
│                                                                   │
│ 7. entrypoints/cli.tsx                                           │
│    - CLI快速路径                                                  │
│    --browser-mcp-server                                          │
│    --browser-native-host                                         │
│                                                                   │
│ 8. skills/bundled/browser.ts                                     │
│    - Skill注册                                                    │
│    - 提示词                                                       │
│                                                                   │
│ 9. hooks/useBrowserNotification.tsx                              │
│    - 启动通知                                                     │
│                                                                   │
│ 10. hooks/usePromptsFromBrowser.tsx                              │
│    - 接收扩展prompt                                               │
└──────────────────────────────────────────────────────────────────┘
```

---

## 附录

### A. Native Messaging协议规范

**消息格式：**
```
[4字节长度（小端序uint32）][UTF-8编码的JSON消息]
```

**最大消息大小：**
```typescript
const MAX_MESSAGE_SIZE = 1024 * 1024  // 1MB
```

**发送示例：**
```typescript
function sendMessage(msg: object) {
  const json = JSON.stringify(msg)
  const bytes = Buffer.from(json, 'utf-8')
  const header = Buffer.alloc(4)
  header.writeUInt32LE(bytes.length, 0)
  process.stdout.write(Buffer.concat([header, bytes]))
}
```

**接收示例：**
```typescript
function readMessage(buffer: Buffer): { message: object, remaining: Buffer } | null {
  if (buffer.length < 4) return null
  const length = buffer.readUInt32LE(0)
  if (buffer.length < 4 + length) return null
  const json = buffer.slice(4, 4 + length).toString('utf-8')
  return { message: JSON.parse(json), remaining: buffer.slice(4 + length) }
}
```

### B. 支持的浏览器配置详情

```typescript
// macOS路径模板
~/Library/Application Support/[Browser]/User Data
~/Library/Application Support/[Browser]/NativeMessagingHosts

// Linux路径模板
~/.config/[browser-name]
~/.config/[browser-name]/NativeMessagingHosts

// Windows路径模板
%LOCALAPPDATA%\[Browser]\User Data
注册表: HKCU\Software\[Browser]\NativeMessagingHosts

// Opera特殊：使用Roaming而非Local
%APPDATA%\Opera Software\Opera Stable
```

### C. 命令行标志一览

| 标志 | 用途 |
|------|------|
| `--chrome` | 强制启用Chrome集成 |
| `--no-chrome` | 强制禁用Chrome集成 |
| `--claude-in-chrome-mcp` | 运行MCP Server子进程 |
| `--chrome-native-host` | 运行Native Host子进程 |

### D. 环境变量

| 变量 | 用途 |
|------|------|
| `CLAUDE_CODE_ENABLE_CFC` | 启用Chrome集成 |
| `CLAUDE_CHROME_PERMISSION_MODE` | 权限模式 (ask/skip_all_permission_checks/follow_a_plan) |
| `LOCAL_BRIDGE` | 使用本地Bridge (localhost:8765) |
| `USE_STAGING_OAUTH` | 使用Staging Bridge |

---

*文档生成日期: 2026-04-02*