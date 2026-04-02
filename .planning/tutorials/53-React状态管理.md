# React状态管理 - 逐行细粒度分析

**分析日期:** 2026-04-02

---

## 一、状态管理概述

Claude Code CLI 采用**自建轻量级状态管理系统**，而非使用 Redux/MobX 等第三方库。核心设计理念：

### 1.1 架构特点

```
┌─────────────────────────────────────────────────────────┐
│                    AppStateProvider                      │
│  ┌───────────────────────────────────────────────────┐  │
│  │              AppStoreContext.Provider              │  │
│  │  ┌─────────────────────────────────────────────┐  │  │
│  │  │              Store<AppState>                │  │  │
│  │  │  ┌───────────────────────────────────────┐  │  │  │
│  │  │  │  getState() → AppState               │  │  │  │
│  │  │  │  setState(updater) → void            │  │  │  │
│  │  │  │  subscribe(listener) → unsubscribe   │  │  │  │
│  │  │  └───────────────────────────────────────┘  │  │  │
│  │  └─────────────────────────────────────────────┘  │  │
│  │                    ↓ ↓ ↓                         │  │
│  │  ┌─────────────────────────────────────────────┐  │  │
│  │  │  useAppState(selector) → slice             │  │  │
│  │  │  useSetAppState() → setState               │  │  │
│  │  │  useAppStateStore() → store                │  │  │
│  │  └─────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ MailboxProv. │  │ VoiceProvider│  │ModalContext │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### 1.2 设计原则

| 原则 | 实现方式 | 文件位置 |
|------|----------|----------|
| **单一数据源** | Store<T> 模式 | `state/store.ts` |
| **不可变更新** | `setState(prev => {...prev, ...})` | `state/AppState.tsx` |
| **选择器模式** | `useAppState(s => s.verbose)` | `state/AppState.tsx:142` |
| **副作用分离** | `onChangeAppState` 回调 | `state/onChangeAppState.ts` |
| **Context 避免嵌套** | `HasAppStateContext` 检查 | `state/AppState.tsx:36` |

---

## 二、state/store.ts - 核心存储实现

### 2.1 类型定义（逐行分析）

```typescript
// 第1-2行：监听器和变更回调类型
type Listener = () => void                        // 无参数通知函数
type OnChange<T> = (args: { newState: T; oldState: T }) => void  // 新旧状态对比
```

**设计意图：**
- `Listener` 简化订阅，仅通知"有变化"，不传递具体数据
- `OnChange` 提供新旧状态对比，用于副作用处理

```typescript
// 第4-8行：Store 接口定义
export type Store<T> = {
  getState: () => T                               // 同步读取当前状态
  setState: (updater: (prev: T) => => T) => void  // 函数式更新
  subscribe: (listener: Listener) => () => void   // 返回取消订阅函数
}
```

**关键设计：**
- `setState` 接收 updater 函数，而非直接值，确保不可变更新
- `subscribe` 返回 unsubscribe 函数，便于 useEffect 清理

### 2.2 createStore 实现（逐行分析）

```typescript
// 第10-13行：函数签名
export function createStore<T>(
  initialState: T,
  onChange?: OnChange<T>,           // 可选副作用回调
): Store<T>
```

```typescript
// 第14-15行：内部状态
let state = initialState           // 私有变量，外部无法直接修改
const listeners = new Set<Listener>()  // Set 避免重复订阅
```

**为什么用 Set：**
- 自动去重，防止同一组件多次订阅
- O(1) 添加/删除性能

```typescript
// 第17-18行：getState 实现
getState: () => state              // 简单返回，无任何处理
```

**为什么不缓存：**
- state 是闭包私有变量，每次访问都是最新值
- 同步读取，无异步开销

```typescript
// 第20-27行：setState 实现
setState: (updater: (prev: T) => => T) => {
  const prev = state               // 保存旧状态用于对比
  const next = updater(prev)       // 执行更新函数
  if (Object.is(next, prev)) return  // 【关键】浅对比优化
  state = next                     // 更新内部状态
  onChange?.({ newState: next, oldState: prev })  // 触发副作用
  for (const listener of listeners) listener()    // 通知所有订阅者
}
```

**逐行关键点：**

| 行号 | 代码 | 作用 |
|------|------|------|
| 21 | `const prev = state` | 捕获旧状态用于对比和副作用 |
| 22 | `const next = updater(prev)` | 函数式更新，确保不可变 |
| 23 | `if (Object.is(next, prev)) return` | **性能优化：相同引用跳过** |
| 24 | `state = next` | 更新闭包变量 |
| 25 | `onChange?.({...})` | 可选副作用（如持久化） |
| 26 | `for (...) listener()` | 批量通知订阅者 |

**Object.is 对比优化：**
```typescript
// 不会触发更新的情况
setState(prev => prev)                  // 返回原对象 → 跳过
setState(prev => { ...prev })           // 浅拷贝但字段未变 → 仍会触发（引用变了）

// 会触发更新的情况
setState(prev => ({ ...prev, verbose: true }))  // 字段变化
```

```typescript
// 第29-32行：subscribe 实现
subscribe: (listener: Listener) => {
  listeners.add(listener)          // 添加到 Set
  return () => listeners.delete(listener)  // 返回取消函数
}
```

**为什么返回函数：**
```typescript
// 典型使用场景
useEffect(() => {
  const unsubscribe = store.subscribe(() => forceRender())
  return unsubscribe  // cleanup 时自动取消
}, [store])
```

### 2.3 完整实现流程图

```
用户调用 setState(updater)
        ↓
    updater(prev)
        ↓
    Object.is(next, prev)?
    ├─ Yes → 直接返回（无变化）
    └─ No → 继续
        ↓
    state = next（更新内部状态）
        ↓
    onChange?.({ newState, oldState })（副作用）
        ↓
    for (listener of listeners) listener()（通知订阅者）
        ↓
    React useSyncExternalStore 触发重新渲染
```

---

## 三、AppState 定义

### 3.1 类型结构概览

`state/AppStateStore.ts` 定义了超过 **450 行** 的 AppState 类型。核心字段分类：

```typescript
// 第89-452行：AppState 类型定义
export type AppState = DeepImmutable<{
  // 【配置相关】
  settings: SettingsJson          // 用户设置
  verbose: boolean                // 详细模式
  mainLoopModel: ModelSetting     // 模型选择
  
  // 【视图状态】
  expandedView: 'none' | 'tasks' | 'teammates'
  viewSelectionMode: 'none' | 'selecting-agent' | 'viewing-agent'
  footerSelection: FooterItem | null
  
  // 【任务管理】
  tasks: { [taskId: string]: TaskState }  // 统一任务状态
  agentNameRegistry: Map<string, AgentId>  // agent 名称映射
  foregroundedTaskId?: string
  viewingAgentTaskId?: string
  
  // 【权限系统】
  toolPermissionContext: ToolPermissionContext
  
  // 【MCP 插件】
  mcp: {
    clients: MCPServerConnection[]
    tools: Tool[]
    commands: Command[]
    resources: Record<string, ServerResource[]>
    pluginReconnectKey: number    // 触发 MCP 重连的计数器
  }
  
  // 【通知系统】
  notifications: {
    current: Notification | null
    queue: Notification[]
  }
  
  // 【远程协作】
  replBridgeEnabled: boolean      // Bridge 开关
  replBridgeConnected: boolean    // 连接状态
  remoteSessionUrl: string | undefined
  
  // 【推测执行】
  speculation: SpeculationState   // 后台预执行状态
  
  // ... 更多字段
}> & {
  // 非 DeepImmutable 的字段（包含函数类型）
  tasks: { [taskId: string]: TaskState }
  agentNameRegistry: Map<string, AgentId>
}
```

### 3.2 DeepImmutable 包装

```typescript
// 第24行：DeepImmutable 类型导入
import type { DeepImmutable } from '../types/utils.js'
```

**作用：** 将所有字段标记为只读，防止直接修改：
```typescript
// 错误用法（TypeScript 报错）
appState.verbose = true  // ❌ 不能直接赋值

// 正确用法
setAppState(prev => ({ ...prev, verbose: true }))  // ✓
```

### 3.3 特殊字段设计

#### SpeculationState（推测执行）

```typescript
// 第58-78行：推测状态类型
export type SpeculationState =
  | { status: 'idle' }             // 空闲状态
  | {
      status: 'active'             // 活跃状态
      id: string                   // 推测 ID
      abort: () => void            // 取消函数
      startTime: number            // 开始时间
      messagesRef: { current: Message[] }  // 可变消息引用
      writtenPathsRef: { current: Set<string> }  // 可变路径集合
      boundary: CompletionBoundary | null
      suggestionLength: number
      toolUseCount: number
      isPipelined: boolean
      contextRef: { current: REPLHookContext }
      pipelinedSuggestion?: {...}
    }
```

**设计亮点：**
- 使用 `Ref` 对象存储可变数据，避免每次消息触发状态更新
- 分离 `status: 'idle' | 'active'`，便于类型判断

#### CompletionBoundary（完成边界）

```typescript
// 第41-51行：完成边界类型
export type CompletionBoundary =
  | { type: 'complete'; completedAt: number; outputTokens: number }
  | { type: 'bash'; command: string; completedAt: number }
  | { type: 'edit'; toolName: string; filePath: string; completedAt: number }
  | { type: 'denied_tool'; toolName: string; detail: string; completedAt: number }
```

**用途：** 标记推测执行的终止点，用于计算节省时间。

### 3.4 getDefaultAppState 初始化

```typescript
// 第456-569行：默认状态初始化
export function getDefaultAppState(): AppState {
  // 第459-462行：延迟导入避免循环依赖
  const teammateUtils = require('../utils/teammate.js')
  
  // 第463-466行：确定初始权限模式
  const initialMode: PermissionMode =
    teammateUtils.isTeammate() && teammateUtils.isPlanModeRequired()
      ? 'plan'
      : 'default'

  return {
    // 第469行：设置初始化
    settings: getInitialSettings(),
    
    // 第470-471行：任务和 agent 注册表
    tasks: {},
    agentNameRegistry: new Map(),
    
    // 第472-476行：核心状态
    verbose: false,
    mainLoopModel: null,
    mainLoopModelForSession: null,
    statusLineText: undefined,
    
    // 第477-482行：视图状态
    expandedView: 'none',
    isBriefOnly: false,
    selectedIPAgentIndex: -1,
    coordinatorTaskIndex: -1,
    viewSelectionMode: 'none',
    footerSelection: null,
    
    // 第483-498行：远程连接状态
    remoteSessionUrl: undefined,
    remoteConnectionStatus: 'connecting',
    replBridgeEnabled: false,
    replBridgeConnected: false,
    // ... 更多 bridge 相关状态
    
    // 第500-503行：权限上下文
    toolPermissionContext: {
      ...getEmptyToolPermissionContext(),
      mode: initialMode,
    },
    
    // 第512-518行：MCP 状态
    mcp: {
      clients: [],
      tools: [],
      commands: [],
      resources: {},
      pluginReconnectKey: 0,
    },
    
    // 第519-529行：插件状态
    plugins: {
      enabled: [],
      disabled: [],
      commands: [],
      errors: [],
      installationStatus: { marketplaces: [], plugins: [] },
      needsRefresh: false,
    },
    
    // 第532-538行：通知和请求队列
    notifications: { current: null, queue: [] },
    elicitation: { queue: [] },
    
    // 第539-541行：思考和提示建议
    thinkingEnabled: shouldEnableThinkingByDefault(),
    promptSuggestionEnabled: shouldEnablePromptSuggestion(),
    
    // 第558-559行：推测执行状态
    speculation: IDLE_SPECULATION_STATE,
    speculationSessionTimeSavedMs: 0,
    
    // 第567行：活跃 overlay 集合
    activeOverlays: new Set<string>(),
    
    // ... 更多初始化
  }
}
```

**关键初始化逻辑：**

| 字段 | 初始化策略 | 说明 |
|------|------------|------|
| `settings` | `getInitialSettings()` | 从配置文件加载 |
| `toolPermissionContext.mode` | 条件判断 | teammate 可能需要 plan 模式 |
| `tasks` | `{}` | 空对象，运行时填充 |
| `speculation` | `IDLE_SPECULATION_STATE` | 预定义常量，避免重复创建 |
| `activeOverlays` | `new Set<string>()` | 空集合，组件注册 overlay |

---

## 四、Context Provider 实现

### 4.1 AppStateProvider（核心）

位置：`state/AppState.tsx:37-109`

```typescript
// 第27-28行：创建 Context
export const AppStoreContext = React.createContext<AppStateStore | null>(null)

// 第36行：嵌套检测 Context
const HasAppStateContext = React.createContext<boolean>(false)
```

**双重 Context 设计：**
- `AppStoreContext`：存储 store 实例
- `HasAppStateContext`：检测嵌套（防止多个 Provider）

```typescript
// 第37-44行：组件定义
export function AppStateProvider(t0) {
  const $ = _c(13)                  // React Compiler 运行时缓存
  const { children, initialState, onChangeAppState } = t0
  
  // 第44-47行：嵌套检测
  const hasAppStateContext = useContext(HasAppStateContext)
  if (hasAppStateContext) {
    throw new Error("AppStateProvider can not be nested within another AppStateProvider")
  }
```

**为什么禁止嵌套：**
- AppState 是全局唯一状态，嵌套会导致状态不一致
- 选择器可能读取错误的 store

```typescript
// 第48-57行：创建 store（带缓存）
let t1
if ($[0] !== initialState || $[1] !== onChangeAppState) {
  t1 = () => createStore(initialState ?? getDefaultAppState(), onChangeAppState)
  $[0] = initialState
  $[1] = onChangeAppState
  $[2] = t1
} else {
  t1 = $[2]
}
const [store] = useState(t1)        // store 在组件生命周期内稳定
```

**缓存策略：**
- `initialState` 或 `onChangeAppState` 变化时重新创建 createStore 函数
- 但 `useState` 确保只执行一次（首次渲染）

```typescript
// 第58-81行：mount 时检查 bypass 权限模式
useEffect(() => {
  const { toolPermissionContext } = store.getState()
  if (
    toolPermissionContext.isBypassPermissionsModeAvailable &&
    isBypassPermissionsModeDisabled()
  ) {
    logForDebugging("Disabling bypass permissions mode on mount...")
    store.setState(prev => ({
      ...prev,
      toolPermissionContext: createDisabledBypassPermissionsContext(prev.toolPermissionContext)
    }))
  }
}, [])
```

**处理竞态条件：**
- 远程设置可能在 Provider mount 前加载
- effect 检查并同步状态

```typescript
// 第83-91行：设置变更监听
const onSettingsChange = useEffectEvent(source => 
  applySettingsChange(source, store.setState)
)
useSettingsChange(onSettingsChange)
```

**useEffectEvent 用途：**
- 创建稳定的回调，不需要放入依赖数组
- 避免因依赖变化重新订阅

```typescript
// 第93-109行：返回 Provider 树
return (
  <HasAppStateContext.Provider value={true}>
    <AppStoreContext.Provider value={store}>
      <MailboxProvider>
        <VoiceProvider>{children}</VoiceProvider>
      </MailboxProvider>
    </AppStoreContext.Provider>
  </HasAppStateContext.Provider>
)
```

**Provider 层级：**
```
HasAppStateContext.Provider（嵌套检测）
  └ AppStoreContext.Provider（核心状态）
      └ MailboxProvider（消息队列）
          └ VoiceProvider（语音状态）
              └ children
```

### 4.2 MailboxProvider（消息队列）

位置：`context/mailbox.tsx`

```typescript
// 第4行：创建 Context
const MailboxContext = createContext<Mailbox | undefined>(undefined)

// 第8-30行：Provider 实现
export function MailboxProvider({ children }: Props) {
  // 第20行：useMemo 创建单例
  const mailbox = useMemo(() => new Mailbox(), [])
  
  return (
    <MailboxContext.Provider value={mailbox}>
      {children}
    </MailboxContext.Provider>
  )
}
```

**设计特点：**
- `Mailbox` 是独立类，不依赖 React 状态
- `useMemo` 确保组件生命周期内唯一实例

```typescript
// 第31-37行：使用 hook
export function useMailbox() {
  const mailbox = useContext(MailboxContext)
  if (!mailbox) {
    throw new Error("useMailbox must be used within a MailboxProvider")
  }
  return mailbox
}
```

### 4.3 VoiceProvider（语音状态）

位置：`context/voice.tsx`

```typescript
// 第10-17行：VoiceState 类型
export type VoiceState = {
  voiceState: 'idle' | 'recording' | 'processing'
  voiceError: string | null
  voiceInterimTranscript: string
  voiceAudioLevels: number[]
  voiceWarmingUp: boolean
}

// 第11-17行：默认状态
const DEFAULT_STATE: VoiceState = {
  voiceState: 'idle',
  voiceError: null,
  voiceInterimTranscript: '',
  voiceAudioLevels: [],
  voiceWarmingUp: false
}
```

```typescript
// 第23-42行：Provider 实现
export function VoiceProvider({ children }: Props) {
  const [store] = useState(() => createStore(DEFAULT_STATE))
  
  return (
    <VoiceContext.Provider value={store}>
      {children}
    </VoiceContext.Provider>
  )
}
```

**与 AppStateProvider 相同模式：**
- 使用 `createStore` 创建独立状态存储
- 提供 `useVoiceState(selector)` 和 `useSetVoiceState()`

```typescript
// 第55-68行：选择器 hook
export function useVoiceState<T>(selector: (state: VoiceState) => T): T {
  const store = useVoiceStore()
  const get = () => selector(store.getState())
  return useSyncExternalStore(store.subscribe, get, get)
}

// 第76-78行：setter hook
export function useSetVoiceState() {
  return useVoiceStore().setState
}

// 第85-87行：同步读取 hook
export function useGetVoiceState() {
  return useVoiceStore().getState
}
```

**三种 hook 用途：**

| Hook | 用途 | 触发渲染？ |
|------|------|-----------|
| `useVoiceState` | 订阅状态切片 | 是（值变化时） |
| `useSetVoiceState` | 获取 setter | 否（稳定引用） |
| `useGetVoiceState` | 同步读取最新值 | 否（仅读取） |

### 4.4 ModalContext（模态框尺寸）

位置：`context/modalContext.tsx`

```typescript
// 第22-26行：Modal Context 类型
type ModalCtx = {
  rows: number                    // 可用行数
  columns: number                 // 可用列数
  scrollRef: RefObject<ScrollBoxHandle | null> | null  // 滚动控制
}

// 第27行：创建 Context
export const ModalContext = createContext<ModalCtx | null>(null)
```

**用途：**
- Slash 命令对话框尺寸受限
- 组件需要知道实际可用空间

```typescript
// 第38-54行：尺寸获取 hook
export function useModalOrTerminalSize(fallback) {
  const ctx = useContext(ModalContext)
  return ctx 
    ? { rows: ctx.rows, columns: ctx.columns }  // 在 Modal 内
    : fallback                                   // 在 Modal 外
}
```

---

## 五、Hooks 集成

### 5.1 useAppState - 选择器模式

位置：`state/AppState.tsx:142-163`

```typescript
// 第142行：函数签名
export function useAppState<T>(selector: (state: AppState) => T): T
```

**选择器设计：**
- 接收 `(state: AppState) => T` 函数
- 仅当选择结果变化时重新渲染

```typescript
// 第143-162行：实现
export function useAppState(selector) {
  const $ = _c(3)                  // Compiler 缓存
  const store = useAppStore()      // 获取 store
  
  let t0
  if ($[0] !== selector || $[1] !== store) {
    t0 = () => {
      const state = store.getState()
      const selected = selector(state)
      
      // 第150-153行：开发环境检查
      if (false && state === selected) {  // 当前禁用
        throw new Error(`Your selector returned the original state...`)
      }
      return selected
    }
    $[0] = selector
    $[1] = store
    $[2] = t0
  } else {
    t0 = $[2]
  }
  const get = t0
  
  // 第162行：核心 - useSyncExternalStore
  return useSyncExternalStore(store.subscribe, get, get)
}
```

**useSyncExternalStore 作用：**
- React 18 引入，用于订阅外部状态
- 参数：`subscribe`, `getSnapshot`, `getServerSnapshot`
- 自动处理订阅/取消订阅

```typescript
// 使用示例
const verbose = useAppState(s => s.verbose)
const model = useAppState(s => s.mainLoopModel)

// 错误示例 - 每次渲染创建新对象
const { text, promptId } = useAppState(s => ({  // ❌ 引用总是新
  text: s.promptSuggestion.text,
  promptId: s.promptSuggestion.promptId
}))

// 正确示例 - 选择现有引用
const { text, promptId } = useAppState(s => s.promptSuggestion)  // ✓
```

### 5.2 useSetAppState - 更新器

位置：`state/AppState.tsx:170-172`

```typescript
export function useSetAppState() {
  return useAppStore().setState
}
```

**设计意图：**
- 返回稳定的 `setState` 函数
- 组件仅更新状态但不订阅变化时使用

```typescript
// 使用示例
function SubmitButton() {
  const setAppState = useSetAppState()  // 不订阅，不触发渲染
  
  const handleSubmit = () => {
    setAppState(prev => ({ ...prev, statusLineText: 'Submitting...' }))
  }
  
  return <button onClick={handleSubmit}>Submit</button>
}
```

### 5.3 useAppStateStore - 直接访问

位置：`state/AppState.tsx:177-179`

```typescript
export function useAppStateStore() {
  return useAppStore()
}
```

**用途：**
- 传递给非 React 代码
- 在 useEffect 中同步读取状态

```typescript
// 使用示例
function useManageMCPConnections() {
  const store = useAppStateStore()
  
  useEffect(() => {
    const { mcp } = store.getState()  // 同步读取
    connectToMCP(mcp.clients)
    
    return store.subscribe(() => {
      // 外部状态变化时的处理
    })
  }, [store])
}
```

### 5.4 useAppStateMaybeOutsideOfProvider

位置：`state/AppState.tsx:186-199`

```typescript
const NOOP_SUBSCRIBE = () => () => {}  // 空订阅函数

export function useAppStateMaybeOutsideOfProvider<T>(
  selector: (state: AppState) => T,
): T | undefined {
  const store = useContext(AppStoreContext)
  
  // 第191行：如果没有 store，返回 undefined
  const get = () => store ? selector(store.getState()) : undefined
  
  return useSyncExternalStore(
    store ? store.subscribe : NOOP_SUBSCRIBE,  // 无 store 时用空订阅
    get
  )
}
```

**用途：**
- 组件可能在 Provider 外渲染
- 测试环境中可能缺少完整 Context 树

### 5.5 useRegisterOverlay - Overlay 注册

位置：`context/overlayContext.tsx:38-104`

```typescript
export function useRegisterOverlay(id: string, enabled = true): void {
  const store = useContext(AppStoreContext)
  const setAppState = store?.setState
  
  useEffect(() => {
    if (!enabled || !setAppState) return
    
    // 第50-60行：注册 overlay
    setAppState(prev => {
      if (prev.activeOverlays.has(id)) return prev  // 已存在
      const next = new Set(prev.activeOverlays)
      next.add(id)
      return { ...prev, activeOverlays: next }
    })
    
    // 第61-73行：清理 - 取消注册
    return () => {
      setAppState(prev => {
        if (!prev.activeOverlays.has(id)) return prev
        const next = new Set(prev.activeOverlays)
        next.delete(id)
        return { ...prev, activeOverlays: next }
      })
    }
  }, [id, enabled, setAppState])
  
  // 第86-103行：强制刷新下一帧
  useLayoutEffect(() => {
    if (!enabled) return
    return () => instances.get(process.stdout)?.invalidatePrevFrame()
  }, [enabled])
}
```

**设计意图：**
- overlay 关闭时，Ink 的 diff 可能残留 ghost 内容
- `invalidatePrevFrame()` 强制全量重绘

---

## 六、最佳实践

### 6.1 选择器优化

```typescript
// ❌ 错误 - 每次返回新对象
const taskCount = useAppState(s => ({
  total: Object.keys(s.tasks).length,
  running: Object.values(s.tasks).filter(t => t.status === 'running').length
}))

// ✓ 正确 - 分别选择
const total = useAppState(s => Object.keys(s.tasks).length)
const running = useAppState(s => 
  Object.values(s.tasks).filter(t => t.status === 'running').length
)

// ✓ 正确 - 选择现有子对象
const mcpTools = useAppState(s => s.mcp.tools)  // 返回稳定引用
```

### 6.2 批量更新

```typescript
// ❌ 错误 - 多次更新
setAppState(prev => ({ ...prev, verbose: true }))
setAppState(prev => ({ ...prev, expandedView: 'tasks' }))

// ✓ 正确 - 单次批量更新
setAppState(prev => ({
  ...prev,
  verbose: true,
  expandedView: 'tasks'
}))
```

### 6.3 副作用分离

```typescript
// 所有副作用集中在 onChangeAppState
// state/onChangeAppState.ts

export function onChangeAppState({ newState, oldState }) {
  // 权限模式同步到 CCR
  if (oldState.toolPermissionContext.mode !== newState.toolPermissionContext.mode) {
    notifySessionMetadataChanged({ permission_mode: ... })
  }
  
  // 模型设置持久化
  if (newState.mainLoopModel !== oldState.mainLoopModel) {
    updateSettingsForSource('userSettings', { model: newState.mainLoopModel })
  }
  
  // 缓存清理
  if (newState.settings !== oldState.settings) {
    clearApiKeyHelperCache()
    clearAwsCredentialsCache()
  }
}
```

**不要在组件中直接执行副作用：**
```typescript
// ❌ 错误 - 副作用散落在组件
const setVerbose = (v: boolean) => {
  setAppState(prev => ({ ...prev, verbose: v }))
  saveGlobalConfig({ verbose: v })  // 直接持久化
}

// ✓ 正确 - 副作用由 onChangeAppState 处理
const setVerbose = (v: boolean) => {
  setAppState(prev => ({ ...prev, verbose: v }))
  // onChangeAppState 自动调用 saveGlobalConfig
}
```

### 6.4 Context 嵌套避免

```typescript
// ❌ 错误 - 嵌套 Provider
<AppStateProvider>
  <AppStateProvider>  // 抛出 Error
    <App />
  </AppStateProvider>
</AppStateProvider>

// ✓ 正确 - 单层 Provider
<AppStateProvider>
  <App />
</AppStateProvider>
```

### 6.5 Store 直接传递

```typescript
// ✓ 正确 - 传递给非 React 代码
function setupBridge(store: AppStateStore) {
  bridge.onMessage(msg => {
    store.setState(prev => ({
      ...prev,
      inbox: { ...prev.inbox, messages: [...prev.inbox.messages, msg] }
    }))
  })
}

// 在组件中获取
function BridgeSetup() {
  const store = useAppStateStore()
  useEffect(() => setupBridge(store), [store])
}
```

---

## 七、从零实现

### 7.1 最小化 Store

```typescript
// 核心实现（15行）
type Listener = () => void

export type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}

export function createStore<T>(initialState: T): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()

  return {
    getState: () => state,
    setState: updater => {
      const next = updater(state)
      if (Object.is(next, state)) return
      state = next
      listeners.forEach(l => l())
    },
    subscribe: listener => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    },
  }
}
```

### 7.2 最小化 Provider

```typescript
import React, { createContext, useContext, useState, useSyncExternalStore } from 'react'

const StoreContext = createContext<Store<AppState> | null>(null)

export function AppStateProvider({ children }: { children: React.ReactNode }) {
  const [store] = useState(() => createStore(getDefaultAppState()))
  return (
    <StoreContext.Provider value={store}>
      {children}
    </StoreContext.Provider>
  )
}

function useStore() {
  const store = useContext(StoreContext)
  if (!store) throw new Error('Missing AppStateProvider')
  return store
}

export function useAppState<T>(selector: (s: AppState) => T): T {
  const store = useStore()
  const get = () => selector(store.getState())
  return useSyncExternalStore(store.subscribe, get, get)
}

export function useSetAppState() {
  return useStore().setState
}
```

### 7.3 带副作用的 Store

```typescript
export function createStore<T>(
  initialState: T,
  onChange?: (args: { newState: T; oldState: T }) => void
): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()

  return {
    getState: () => state,
    setState: updater => {
      const prev = state
      const next = updater(prev)
      if (Object.is(next, prev)) return
      state = next
      onChange?.({ newState: next, oldState: prev })
      listeners.forEach(l => l())
    },
    subscribe: listener => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    },
  }
}

// 使用
const store = createStore(initialState, ({ newState, oldState }) => {
  if (newState.token !== oldState.token) {
    localStorage.setItem('token', newState.token)
  }
})
```

### 7.4 选择器缓存优化

```typescript
// 高级实现：带缓存的选择器
function useAppStateWithCache<T>(
  selector: (s: AppState) => T,
  deps: React.DependencyList = []
): T {
  const store = useStore()
  const cacheRef = useRef<T>()
  
  const getSnapshot = useCallback(() => {
    const next = selector(store.getState())
    // 仅当依赖变化或值变化时更新缓存
    if (cacheRef.current === undefined || !Object.is(next, cacheRef.current)) {
      cacheRef.current = next
    }
    return cacheRef.current
  }, [store, ...deps])
  
  return useSyncExternalStore(store.subscribe, getSnapshot, getSnapshot)
}
```

---

## 附录：文件索引

| 文件 | 主要内容 |
|------|----------|
| `state/store.ts` | Store<T> 类型定义和 createStore 实现 |
| `state/AppStateStore.ts` | AppState 类型定义（450+行）和 getDefaultAppState |
| `state/AppState.tsx` | AppStateProvider、useAppState 等 hooks |
| `state/onChangeAppState.ts` | 状态变更副作用处理 |
| `state/selectors.ts` | 纯函数选择器 |
| `state/teammateViewHelpers.ts` | teammate 视图辅助函数 |
| `context/mailbox.tsx` | MailboxProvider |
| `context/modalContext.tsx` | Modal 尺寸 Context |
| `context/notifications.tsx` | 通知队列 hooks |
| `context/overlayContext.tsx` | Overlay 注册 hooks |
| `context/voice.tsx` | VoiceProvider 和语音状态 hooks |

---

*分析完成：2026-04-02*