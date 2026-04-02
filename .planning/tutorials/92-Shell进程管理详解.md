# Shell 进程管理详解

**文档日期:** 2026-04-02

> **前置知识:** 阅读本文前，建议先阅读:
> - [27-BashTool工具详解](./27-BashTool工具详解.md) - BashTool 的上层调用
> - [39-沙箱隔离系统](./39-沙箱隔离系统.md) - 安全隔离机制

---

## 目录

1. [概述](#1-概述)
2. [Shell.ts 核心实现](#2-shellts-核心实现)
3. [ShellCommand.ts 命令封装](#3-shellcommandts-命令封装)
4. [进程生命周期](#4-进程生命周期)
5. [错误处理与恢复](#5-错误处理与恢复)
6. [与 BashTool 的关系](#6-与-bashtool-的关系)

---

## 1. 概述

### 1.1 Shell 进程管理的定位

Shell 进程管理模块是 BashTool 的底层依赖，负责：

- **进程创建与管理** - 创建子进程执行命令
- **输入输出流处理** - 管理 stdin/stdout/stderr
- **信号处理** - 处理中断、终止等信号
- **资源清理** - 确保进程正确退出

### 1.2 文件结构

```
utils/
├── Shell.ts           # 核心进程管理
├── ShellCommand.ts    # 命令封装
└── shell/
    ├── shell-quote.ts    # 命令引号处理
    ├── shell-env.ts      # 环境变量
    └── shell-utils.ts    # 工具函数
```

### 1.3 架构位置

```
BashTool (工具层)
    ↓
ShellCommand (命令封装层)
    ↓
Shell (进程管理层)
    ↓
ChildProcess (Node.js 子进程)
```

---

## 2. Shell.ts 核心实现

### 2.1 Shell 类定义

**文件位置:** `utils/Shell.ts`

```typescript
export class Shell {
  private process: ChildProcess | null = null
  private stdout: string = ''
  private stderr: string = ''
  private exitCode: number | null = null

  constructor(private options: ShellOptions) {}

  // 执行命令
  async execute(command: string): Promise<ShellResult>

  // 中断执行
  abort(): void

  // 清理资源
  cleanup(): void
}
```

### 2.2 执行流程

```
execute(command)
    │
    ├─ 1. 解析命令
    │      └─ shell-quote.ts 处理引号和转义
    │
    ├─ 2. 准备环境
    │      ├─ 继承当前环境变量
    │      └─ 添加自定义环境变量
    │
    ├─ 3. 创建子进程
    │      └─ spawn(shell, ['-c', command], options)
    │
    ├─ 4. 设置流处理
    │      ├─ stdout → 累加器
    │      ├─ stderr → 累加器
    │      └─ stdin → 可选输入
    │
    ├─ 5. 等待完成
    │      ├─ 正常退出 → exitCode = 0
    │      ├─ 错误退出 → exitCode = 1-255
    │      └─ 信号终止 → exitCode = null
    │
    └─ 6. 返回结果
           └─ { stdout, stderr, exitCode }
```

### 2.3 进程选项

```typescript
interface ShellOptions {
  // 工作目录
  cwd?: string

  // 环境变量
  env?: Record<string, string>

  // 超时时间（毫秒）
  timeout?: number

  // 输入内容
  input?: string

  // 最大缓冲区大小
  maxBuffer?: number

  // 信号处理
  signal?: AbortSignal
}
```

### 2.4 输出流处理

```typescript
// 使用累加器处理大输出
class OutputAccumulator {
  private chunks: Buffer[] = []
  private totalSize: number = 0

  append(chunk: Buffer): void {
    this.chunks.push(chunk)
    this.totalSize += chunk.length
  }

  toString(maxSize?: number): string {
    const buffer = Buffer.concat(this.chunks)
    if (maxSize && buffer.length > maxSize) {
      // 截断处理
      return buffer.slice(-maxSize).toString('utf-8')
    }
    return buffer.toString('utf-8')
  }
}
```

---

## 3. ShellCommand.ts 命令封装

### 3.1 ShellCommand 类

**文件位置:** `utils/ShellCommand.ts`

```typescript
export class ShellCommand {
  constructor(
    private command: string,
    private args: string[] = [],
    private options: ShellCommandOptions = {}
  ) {}

  // 执行并返回结果
  async run(): Promise<CommandResult>

  // 流式执行，返回 AsyncGenerator
  async *stream(): AsyncGenerator<StreamChunk>

  // 构建命令字符串
  toString(): string
}
```

### 3.2 命令构建

```typescript
// 安全构建命令
function buildCommand(command: string, args: string[]): string {
  // 1. 处理引号
  const escapedArgs = args.map(arg => shellEscape(arg))

  // 2. 构建命令字符串
  return `${command} ${escapedArgs.join(' ')}`
}

// shell 转义
function shellEscape(arg: string): string {
  if (/^[a-zA-Z0-9_\-\.\/]+$/.test(arg)) {
    return arg  // 安全字符，不需要转义
  }
  return `'${arg.replace(/'/g, "'\\''")}'`  // 单引号包裹
}
```

### 3.3 流式执行

```typescript
async *stream(): AsyncGenerator<StreamChunk> {
  const shell = new Shell(this.options)

  // 开始执行
  const process = shell.execute(this.toString())

  // 产生输出块
  for await (const chunk of process.stdout) {
    yield { type: 'stdout', data: chunk }
  }

  for await (const chunk of process.stderr) {
    yield { type: 'stderr', data: chunk }
  }

  // 最终结果
  const result = await process
  yield { type: 'exit', code: result.exitCode }
}
```

---

## 4. 进程生命周期

### 4.1 状态转换

```
┌─────────┐    execute()    ┌─────────┐
│  Idle   │ ──────────────> │ Running │
└─────────┘                 └────┬────┘
     ▲                           │
     │                           │
     │     ┌─────────────────────┼─────────────────────┐
     │     │                     │                     │
     │     ▼                     ▼                     ▼
     │  ┌─────────┐         ┌─────────┐         ┌─────────┐
     └──│ Success │         │  Error  │         │ Aborted │
        │ exit=0  │         │ exit>0  │         │ signal  │
        └─────────┘         └─────────┘         └─────────┘
```

### 4.2 信号处理

```typescript
// 处理中断信号
process.on('SIGINT', () => {
  // 1. 发送 SIGINT 给子进程
  childProcess.kill('SIGINT')

  // 2. 等待优雅退出
  setTimeout(() => {
    if (childProcess.exitCode === null) {
      // 3. 强制终止
      childProcess.kill('SIGKILL')
    }
  }, 5000)
})
```

### 4.3 超时处理

```typescript
async execute(command: string, timeout: number): Promise<ShellResult> {
  const controller = new AbortController()

  const timeoutId = setTimeout(() => {
    controller.abort()
  }, timeout)

  try {
    return await this.doExecute(command, controller.signal)
  } finally {
    clearTimeout(timeoutId)
  }
}
```

---

## 5. 错误处理与恢复

### 5.1 错误类型

```typescript
// 错误分类
type ShellError =
  | { type: 'spawn'; error: Error }      // 进程创建失败
  | { type: 'timeout'; elapsed: number }  // 执行超时
  | { type: 'signal'; signal: string }    // 信号终止
  | { type: 'exit'; code: number }        // 非零退出码
  | { type: 'abort'; reason: string }     // 用户中断
```

### 5.2 错误恢复策略

| 错误类型 | 恢复策略 |
|:---|:---|
| spawn 失败 | 检查命令是否存在，提示用户安装 |
| 超时 | 可配置重试，或提示用户增加超时时间 |
| 信号终止 | 不重试，返回中断信息 |
| 非零退出码 | 返回 stderr 供上层处理 |

### 5.3 错误传播

```typescript
// 上层调用示例（BashTool）
try {
  const result = await shell.execute(command)
  if (result.exitCode !== 0) {
    // 非零退出码，作为工具错误返回
    return {
      error: result.stderr || `Exit code: ${result.exitCode}`,
      stdout: result.stdout
    }
  }
  return { data: result.stdout }
} catch (error) {
  if (error instanceof TimeoutError) {
    // 超时错误，特殊处理
    return { error: `Command timed out after ${timeout}ms` }
  }
  throw error  // 其他错误向上传播
}
```

---

## 6. 与 BashTool 的关系

### 6.1 调用层次

```
用户输入 → BashTool.execute()
              │
              ├─ checkPermissions() 权限检查
              │
              ├─ applySandbox() 沙箱配置
              │
              └─ ShellCommand.run()
                    │
                    └─ Shell.execute()
                          │
                          └─ child_process.spawn()
```

### 6.2 职责划分

| 模块 | 职责 |
|:---|:---|
| **BashTool** | 工具接口、权限检查、结果格式化 |
| **ShellCommand** | 命令构建、参数转义、流式输出 |
| **Shell** | 进程创建、信号处理、资源清理 |

### 6.3 数据流

```
BashTool
    │
    │ command: string, options: BashToolOptions
    ▼
ShellCommand
    │
    │ commandString: string, shellOptions: ShellOptions
    ▼
Shell
    │
    │ { stdout: string, stderr: string, exitCode: number }
    ▼
ShellCommand
    │
    │ { stdout: string, stderr: string, interrupted: boolean }
    ▼
BashTool
    │
    │ ToolResult { data: { stdout, stderr, ... } }
    ▼
模型
```

---

## 附录: 关键文件路径

| 文件 | 职责 |
|:---|:---|
| `utils/Shell.ts` | 核心进程管理类 |
| `utils/ShellCommand.ts` | 命令封装类 |
| `utils/shell/shell-quote.ts` | Shell 引号处理 |
| `utils/shell/shell-env.ts` | 环境变量管理 |
| `tools/BashTool/BashTool.tsx` | BashTool 实现 |

---

*Shell 进程管理详解生成: 2026-04-02*