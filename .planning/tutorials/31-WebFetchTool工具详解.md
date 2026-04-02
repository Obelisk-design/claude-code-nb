# WebFetchTool 工具详解

**分析日期:** 2026-04-02

> **前置知识:** 阅读本文前，建议先阅读:
> - [03-工具系统设计](./03-工具系统设计.md) - 理解Tool接口与buildTool工厂

---

## 1. 概述

### 1.1 WebFetch工具作用

WebFetchTool 是 Claude Code CLI 中用于获取网页内容的核心工具，位于 `tools/WebFetchTool/` 目录。其主要功能包括：

- 从指定URL获取内容
- 将HTML转换为Markdown格式
- 使用AI模型处理网页内容
- 支持用户自定义提示词提取信息

**核心文件结构:**
```
tools/WebFetchTool/
├── WebFetchTool.ts   # 主工具定义
├── UI.tsx            # UI渲染组件
├── preapproved.ts    # 预批准域名列表
├── prompt.ts         # 提示词定义
└── utils.ts          # 核心工具函数
```

### 1.2 网络访问能力

WebFetch工具具备以下网络访问能力：

- **HTTP/HTTPS支持:** 自动将HTTP升级为HTTPS
- **重定向处理:** 支持同域名重定向（包括www变体）
- **跨域重定向检测:** 当重定向到不同域名时，返回重定向信息而非自动跟随
- **内容类型支持:** 
  - `text/html` - 转换为Markdown
  - `text/markdown` - 直接使用
  - 二进制内容（PDF等）- 保存到磁盘并提供摘要

**关键配置常量 (`utils.ts` 第106-128行):**
```typescript
const MAX_URL_LENGTH = 2000                    // URL最大长度
const MAX_HTTP_CONTENT_LENGTH = 10 * 1024 * 1024  // 最大响应体 10MB
const FETCH_TIMEOUT_MS = 60_000                // 请求超时 60秒
const DOMAIN_CHECK_TIMEOUT_MS = 10_000          // 域名检查超时 10秒
const MAX_REDIRECTS = 10                        // 最大重定向次数
export const MAX_MARKDOWN_LENGTH = 100_000     // Markdown最大长度 100K字符
```

---

## 2. 输入Schema

### 2.1 输入参数定义

**文件位置:** `WebFetchTool.ts` 第24-29行

```typescript
const inputSchema = lazySchema(() =>
  z.strictObject({
    url: z.string().url().describe('The URL to fetch content from'),
    prompt: z.string().describe('The prompt to run on the fetched content'),
  }),
)
```

| 参数 | 类型 | 必需 | 描述 |
|------|------|------|------|
| `url` | string (URL) | 是 | 要获取内容的URL，必须是有效的URL格式 |
| `prompt` | string | 是 | 对获取的内容应用的提示词，描述要从页面中提取什么信息 |

### 2.2 输出Schema

**文件位置:** `WebFetchTool.ts` 第32-46行

```typescript
const outputSchema = lazySchema(() =>
  z.object({
    bytes: z.number().describe('Size of the fetched content in bytes'),
    code: z.number().describe('HTTP response code'),
    codeText: z.string().describe('HTTP response code text'),
    result: z.string().describe('Processed result from applying the prompt to the content'),
    durationMs: z.number().describe('Time taken to fetch and process the content'),
    url: z.string().describe('The URL that was fetched'),
  }),
)
```

| 字段 | 类型 | 描述 |
|------|------|------|
| `bytes` | number | 获取内容的字节大小 |
| `code` | number | HTTP响应状态码 |
| `codeText` | string | HTTP状态文本（如 "OK", "Moved Permanently"） |
| `result` | string | AI处理后的结果文本 |
| `durationMs` | number | 获取和处理耗时（毫秒） |
| `url` | string | 实际获取的URL |

### 2.3 超时配置

**请求超时层级:**

1. **主请求超时** (`FETCH_TIMEOUT_MS`): 60秒
2. **域名检查超时** (`DOMAIN_CHECK_TIMEOUT_MS`): 10秒
3. **缓存TTL**: 15分钟

**用户可通过 AbortController 取消请求:**
```typescript
async call({ url, prompt }, { abortController, options: { isNonInteractiveSession } }) {
  const response = await getURLMarkdownContent(url, abortController)
  // ...
}
```

---

## 3. 网络请求实现

### 3.1 HTTP客户端

**文件位置:** `utils.ts`

WebFetch使用 **axios** 作为HTTP客户端，配置如下：

```typescript
// utils.ts 第272-282行
return await axios.get(url, {
  signal,
  timeout: FETCH_TIMEOUT_MS,
  maxRedirects: 0,              // 手动处理重定向
  responseType: 'arraybuffer',  // 支持二进制内容
  maxContentLength: MAX_HTTP_CONTENT_LENGTH,
  headers: {
    Accept: 'text/markdown, text/html, */*',
    'User-Agent': getWebFetchUserAgent(),
  },
})
```

**关键特性:**
- `maxRedirects: 0` - 禁用axios自动重定向，实现自定义安全重定向逻辑
- `responseType: 'arraybuffer'` - 支持二进制内容处理
- `maxContentLength: 10MB` - 防止资源耗尽

### 3.2 重定向处理

**安全重定向策略 (`utils.ts` 第211-243行):**

```typescript
export function isPermittedRedirect(
  originalUrl: string,
  redirectUrl: string,
): boolean {
  try {
    const parsedOriginal = new URL(originalUrl)
    const parsedRedirect = new URL(redirectUrl)

    // 协议必须相同
    if (parsedRedirect.protocol !== parsedOriginal.protocol) {
      return false
    }

    // 端口必须相同
    if (parsedRedirect.port !== parsedOriginal.port) {
      return false
    }

    // 不允许带用户名/密码的重定向
    if (parsedRedirect.username || parsedRedirect.password) {
      return false
    }

    // 允许添加/移除 www. 前缀
    const stripWww = (hostname: string) => hostname.replace(/^www\./, '')
    const originalHostWithoutWww = stripWww(parsedOriginal.hostname)
    const redirectHostWithoutWww = stripWww(parsedRedirect.hostname)
    return originalHostWithoutWww === redirectHostWithoutWww
  } catch (_error) {
    return false
  }
}
```

**允许的重定向场景:**
1. `example.com` → `www.example.com` (添加www)
2. `www.example.com` → `example.com` (移除www)
3. 同域名内的路径变更
4. 以上组合

**禁止的重定向场景:**
1. 跨域重定向 → 返回重定向信息，让用户决定
2. 协议降级 (`https://` → `http://`)
3. 端口变更
4. 包含用户凭证的URL

### 3.3 递归重定向跟随

**文件位置:** `utils.ts` 第262-329行

```typescript
export async function getWithPermittedRedirects(
  url: string,
  signal: AbortSignal,
  redirectChecker: (originalUrl: string, redirectUrl: string) => boolean,
  depth = 0,
): Promise<AxiosResponse<ArrayBuffer> | RedirectInfo> {
  if (depth > MAX_REDIRECTS) {
    throw new Error(`Too many redirects (exceeded ${MAX_REDIRECTS})`)
  }
  
  try {
    return await axios.get(url, { /* ... */ maxRedirects: 0 })
  } catch (error) {
    if (/* 301, 302, 307, 308 重定向 */) {
      const redirectUrl = new URL(redirectLocation, url).toString()
      
      if (redirectChecker(url, redirectUrl)) {
        // 递归跟随安全重定向
        return getWithPermittedRedirects(redirectUrl, signal, redirectChecker, depth + 1)
      } else {
        // 返回重定向信息给调用者
        return { type: 'redirect', originalUrl: url, redirectUrl, statusCode }
      }
    }
    throw error
  }
}
```

### 3.4 错误处理

**自定义错误类型 (`utils.ts` 第21-48行):**

```typescript
// 域名被阻止
class DomainBlockedError extends Error {
  constructor(domain: string) {
    super(`Claude Code is unable to fetch from ${domain}`)
    this.name = 'DomainBlockedError'
  }
}

// 域名检查失败（网络问题）
class DomainCheckFailedError extends Error {
  constructor(domain: string) {
    super(
      `Unable to verify if domain ${domain} is safe to fetch. ` +
      `This may be due to network restrictions...`
    )
    this.name = 'DomainCheckFailedError'
  }
}

// 网络出口代理阻止
class EgressBlockedError extends Error {
  constructor(public readonly domain: string) {
    super(JSON.stringify({
      error_type: 'EGRESS_BLOCKED',
      domain,
      message: `Access to ${domain} is blocked by the network egress proxy.`,
    }))
    this.name = 'EgressBlockedError'
  }
}
```

**错误检测逻辑 (`utils.ts` 第318-328行):**
```typescript
// 检测出口代理阻止
if (
  axios.isAxiosError(error) &&
  error.response?.status === 403 &&
  error.response.headers['x-proxy-error'] === 'blocked-by-allowlist'
) {
  const hostname = new URL(url).hostname
  throw new EgressBlockedError(hostname)
}
```

---

## 4. 内容处理

### 4.1 HTML转Markdown

**文件位置:** `utils.ts` 第85-97行

使用 **turndown** 库进行HTML到Markdown的转换：

```typescript
// 延迟单例模式 - 首次HTML获取时才加载turndown模块（约1.4MB）
let turndownServicePromise: Promise<InstanceType<TurndownCtor>> | undefined

function getTurndownService(): Promise<InstanceType<TurndownCtor>> {
  return (turndownServicePromise ??= import('turndown').then(m => {
    const Turndown = (m as unknown as { default: TurndownCtor }).default
    return new Turndown()
  }))
}
```

**转换逻辑 (`utils.ts` 第454-466行):**
```typescript
let markdownContent: string
let contentBytes: number

if (contentType.includes('text/html')) {
  // HTML内容转换为Markdown
  markdownContent = (await getTurndownService()).turndown(htmlContent)
  contentBytes = Buffer.byteLength(markdownContent)
} else {
  // 非HTML内容直接使用原始文本
  markdownContent = htmlContent
  contentBytes = bytes
}
```

### 4.2 内容提取

**AI处理流程 (`utils.ts` 第484-530行):**

```typescript
export async function applyPromptToMarkdown(
  prompt: string,
  markdownContent: string,
  signal: AbortSignal,
  isNonInteractiveSession: boolean,
  isPreapprovedDomain: boolean,
): Promise<string> {
  // 截断内容以避免"Prompt is too long"错误
  const truncatedContent =
    markdownContent.length > MAX_MARKDOWN_LENGTH
      ? markdownContent.slice(0, MAX_MARKDOWN_LENGTH) +
        '\n\n[Content truncated due to length...]'
      : markdownContent

  const modelPrompt = makeSecondaryModelPrompt(
    truncatedContent,
    prompt,
    isPreapprovedDomain,
  )
  
  // 使用Haiku模型处理
  const assistantMessage = await queryHaiku({
    systemPrompt: asSystemPrompt([]),
    userPrompt: modelPrompt,
    signal,
    options: { querySource: 'web_fetch_apply', /* ... */ },
  })

  // 返回处理结果
  if (signal.aborted) {
    throw new AbortError()
  }

  const { content } = assistantMessage.message
  if (content.length > 0) {
    const contentBlock = content[0]
    if ('text' in contentBlock!) {
      return contentBlock.text
    }
  }
  return 'No response from model'
}
```

### 4.3 Token限制

**内容截断策略:**

1. **最大Markdown长度:** 100,000字符
2. **二级模型:** 使用Claude Haiku（快速、低成本）
3. **预批准域名特殊处理:**

```typescript
// WebFetchTool.ts 第264-278行
if (
  isPreapproved &&
  contentType.includes('text/markdown') &&
  content.length < MAX_MARKDOWN_LENGTH
) {
  // 预批准域名的原始Markdown直接返回，无需AI处理
  result = content
} else {
  // 其他情况需要AI处理
  result = await applyPromptToMarkdown(prompt, content, /* ... */)
}
```

### 4.4 二进制内容处理

**文件位置:** `utils.ts` 第440-449行

```typescript
// 二进制内容保存到磁盘
if (isBinaryContentType(contentType)) {
  const persistId = `webfetch-${Date.now()}-${Math.random().toString(36).slice(2, 8)}`
  const result = await persistBinaryContent(rawBuffer, contentType, persistId)
  if (!('error' in result)) {
    persistedPath = result.filepath
    persistedSize = result.size
  }
}
```

二进制文件（如PDF）会:
1. 保存到临时目录
2. 尝试UTF-8解码提取文本结构
3. 在结果中追加文件路径信息

---

## 5. 用户使用指南

### 5.1 如何获取网页内容

**基本用法:**

WebFetch工具接受两个参数：`url` 和 `prompt`。

```
用户请求: "帮我查看 https://docs.python.org/3/library/os.html 的内容"

Claude会调用:
{
  "url": "https://docs.python.org/3/library/os.html",
  "prompt": "提取页面的主要内容，包括模块功能描述和主要函数列表"
}
```

### 5.2 处理复杂页面

**场景1: 技术文档提取**

```
用户: "从React文档中获取关于useState hook的用法说明"

Claude调用:
{
  "url": "https://react.dev/reference/react/useState",
  "prompt": "提取useState的定义、用法、参数说明和示例代码"
}
```

**场景2: API文档解析**

```
用户: "获取axios库的配置选项文档"

Claude调用:
{
  "url": "https://axios-http.com/docs/req_config",
  "prompt": "列出所有请求配置选项及其说明"
}
```

### 5.3 超时问题解决

**常见超时场景:**

1. **服务器响应慢** - 60秒超时
2. **域名检查超时** - 10秒超时
3. **网络出口代理阻止** - 返回特殊错误

**解决方案:**

```typescript
// 企业用户可跳过域名预检（设置项）
const settings = getSettings_DEPRECATED()
if (!settings.skipWebFetchPreflight) {
  // 执行域名黑名单检查
  const checkResult = await checkDomainBlocklist(hostname)
  // ...
}
```

**用户提示:** 如果遇到域名检查失败，可以：
1. 检查网络连接
2. 确认域名是否被阻止
3. 企业用户可配置 `skipWebFetchPreflight` 设置

### 5.4 最佳实践

**1. 提示词编写建议:**

```typescript
// prompt.ts 第28-34行
const guidelines = isPreapprovedDomain
  ? `Provide a concise response based on the content above. 
     Include relevant details, code examples, and documentation excerpts as needed.`
  : `Provide a concise response based only on the content above. In your response:
   - Enforce a strict 125-character maximum for quotes from any source document.
   - Use quotation marks for exact language from articles.
   - Never produce or reproduce exact song lyrics.`
```

**2. 预批准域名列表:**

预批准域名可以更快处理，直接返回Markdown内容而无需AI摘要。包括：
- 主要编程语言文档（Python, JavaScript, Rust等）
- 框架文档（React, Vue, Angular等）
- 云服务文档（AWS, GCP, Azure等）

**3. 缓存利用:**

```typescript
// 15分钟缓存，50MB大小限制
const CACHE_TTL_MS = 15 * 60 * 1000
const MAX_CACHE_SIZE_BYTES = 50 * 1024 * 1024

// 重复请求相同URL会命中缓存
const cachedEntry = URL_CACHE.get(url)
if (cachedEntry) {
  return cachedEntry // 直接返回缓存内容
}
```

**4. 重定向处理:**

当检测到跨域重定向时，工具会返回特殊格式：

```
REDIRECT DETECTED: The URL redirects to a different host.

Original URL: https://original.com/page
Redirect URL: https://newdomain.com/page
Status: 301 Moved Permanently

To complete your request, I need to fetch content from the redirected URL.
Please use WebFetch again with these parameters:
- url: "https://newdomain.com/page"
- prompt: "your prompt"
```

---

## 6. 安全机制

### 6.1 URL验证

**验证函数 (`utils.ts` 第139-169行):**

```typescript
export function validateURL(url: string): boolean {
  // 1. 长度检查
  if (url.length > MAX_URL_LENGTH) {
    return false
  }

  // 2. URL格式验证
  let parsed
  try {
    parsed = new URL(url)
  } catch {
    return false
  }

  // 3. 禁止用户名/密码
  if (parsed.username || parsed.password) {
    return false
  }

  // 4. 主机名格式检查（必须有点分隔，至少两部分）
  const hostname = parsed.hostname
  const parts = hostname.split('.')
  if (parts.length < 2) {
    return false
  }

  return true
}
```

**安全限制:**
- URL最大长度: 2000字符
- 禁止带凭证的URL（`user:pass@host`）
- 主机名必须是可公开解析的域名

### 6.2 域名白名单

**预批准域名列表 (`preapproved.ts`):**

```typescript
export const PREAPPROVED_HOSTS = new Set([
  // Anthropic相关
  'platform.claude.com',
  'code.claude.com',
  'modelcontextprotocol.io',
  'github.com/anthropics',  // 路径限定
  
  // 编程语言文档
  'docs.python.org',
  'go.dev',
  'doc.rust-lang.org',
  'www.typescriptlang.org',
  
  // 框架文档
  'react.dev',
  'vuejs.org',
  'nextjs.org',
  // ... 更多
])
```

**匹配逻辑 (`preapproved.ts` 第154-166行):**

```typescript
export function isPreapprovedHost(hostname: string, pathname: string): boolean {
  // 快速路径：完整主机名匹配
  if (HOSTNAME_ONLY.has(hostname)) return true
  
  // 路径前缀匹配（如 github.com/anthropics）
  const prefixes = PATH_PREFIXES.get(hostname)
  if (prefixes) {
    for (const p of prefixes) {
      // 严格匹配："/anthropics" 只匹配 "/anthropics" 或 "/anthropics/"
      if (pathname === p || pathname.startsWith(p + '/')) return true
    }
  }
  return false
}
```

### 6.3 SSRF防护

**多层防护机制:**

**1. 域名黑名单预检 (`utils.ts` 第176-203行):**

```typescript
export async function checkDomainBlocklist(
  domain: string,
): Promise<DomainCheckResult> {
  // 缓存检查
  if (DOMAIN_CHECK_CACHE.has(domain)) {
    return { status: 'allowed' }
  }
  
  // 调用Anthropic API检查域名
  const response = await axios.get(
    `https://api.anthropic.com/api/web/domain_info?domain=${encodeURIComponent(domain)}`,
    { timeout: DOMAIN_CHECK_TIMEOUT_MS },
  )
  
  if (response.data.can_fetch === true) {
    DOMAIN_CHECK_CACHE.set(domain, true)
    return { status: 'allowed' }
  }
  return { status: 'blocked' }
}
```

**2. 重定向安全检查:**

防止开放重定向漏洞利用：
- 不自动跟随跨域重定向
- 只允许同域重定向（含www变体）
- 返回重定向信息让用户确认

**3. 内部网络保护:**

```typescript
// 主机名必须有至少两部分（阻止 "localhost", "internal" 等）
const parts = hostname.split('.')
if (parts.length < 2) {
  return false
}
```

**4. 协议限制:**

```typescript
// 自动升级HTTP到HTTPS
if (parsedUrl.protocol === 'http:') {
  parsedUrl.protocol = 'https:'
  upgradedUrl = parsedUrl.toString()
}
```

### 6.4 权限控制

**权限检查流程 (`WebFetchTool.ts` 第104-180行):**

```typescript
async checkPermissions(input, context): Promise<PermissionDecision> {
  // 1. 检查预批准域名
  if (isPreapprovedHost(parsedUrl.hostname, parsedUrl.pathname)) {
    return { behavior: 'allow', /* ... */ }
  }

  // 2. 检查拒绝规则
  const denyRule = getRuleByContentsForTool(permissionContext, WebFetchTool, 'deny')
    .get(ruleContent)
  if (denyRule) {
    return { behavior: 'deny', /* ... */ }
  }

  // 3. 检查询问规则
  const askRule = getRuleByContentsForTool(permissionContext, WebFetchTool, 'ask')
    .get(ruleContent)
  if (askRule) {
    return { behavior: 'ask', /* ... */ }
  }

  // 4. 检查允许规则
  const allowRule = getRuleByContentsForTool(permissionContext, WebFetchTool, 'allow')
    .get(ruleContent)
  if (allowRule) {
    return { behavior: 'allow', /* ... */ }
  }

  // 5. 默认询问用户
  return { behavior: 'ask', /* ... */ }
}
```

**权限规则格式:**
```typescript
// 按域名生成权限规则内容
function webFetchToolInputToPermissionRuleContent(input): string {
  const { url } = parsedInput.data
  const hostname = new URL(url).hostname
  return `domain:${hostname}`  // 例如: "domain:example.com"
}
```

---

## 7. 从零实现

### 7.1 核心实现步骤

**步骤1: 定义工具Schema**

```typescript
import { z } from 'zod/v4'
import { buildTool, type ToolDef } from '../../Tool.js'

const inputSchema = z.strictObject({
  url: z.string().url().describe('The URL to fetch content from'),
  prompt: z.string().describe('The prompt to run on the fetched content'),
})

const outputSchema = z.object({
  bytes: z.number(),
  code: z.number(),
  codeText: z.string(),
  result: z.string(),
  durationMs: z.number(),
  url: z.string(),
})
```

**步骤2: 实现HTTP请求**

```typescript
import axios from 'axios'

const FETCH_TIMEOUT_MS = 60_000
const MAX_HTTP_CONTENT_LENGTH = 10 * 1024 * 1024

async function fetchUrl(url: string, signal: AbortSignal) {
  return await axios.get(url, {
    signal,
    timeout: FETCH_TIMEOUT_MS,
    maxRedirects: 0,  // 手动处理重定向
    responseType: 'arraybuffer',
    maxContentLength: MAX_HTTP_CONTENT_LENGTH,
    headers: {
      Accept: 'text/markdown, text/html, */*',
      'User-Agent': 'Claude-Code-WebFetch/1.0',
    },
  })
}
```

**步骤3: HTML转Markdown**

```typescript
import Turndown from 'turndown'

let turndownService: Turndown | undefined

function getTurndownService(): Turndown {
  return (turndownService ??= new Turndown())
}

async function htmlToMarkdown(html: string): Promise<string> {
  return getTurndownService().turndown(html)
}
```

**步骤4: 内容处理**

```typescript
const MAX_MARKDOWN_LENGTH = 100_000

async function processContent(
  prompt: string,
  markdown: string,
  signal: AbortSignal,
): Promise<string> {
  // 截断过长内容
  const truncated = markdown.length > MAX_MARKDOWN_LENGTH
    ? markdown.slice(0, MAX_MARKDOWN_LENGTH) + '\n\n[Content truncated...]'
    : markdown

  // 使用AI模型处理
  const result = await queryAI({
    prompt: `Web page content:\n---\n${truncated}\n---\n\n${prompt}`,
    signal,
  })
  
  return result
}
```

**步骤5: 缓存实现**

```typescript
import { LRUCache } from 'lru-cache'

const CACHE_TTL_MS = 15 * 60 * 1000  // 15分钟
const MAX_CACHE_SIZE = 50 * 1024 * 1024  // 50MB

const cache = new LRUCache<string, CacheEntry>({
  maxSize: MAX_CACHE_SIZE,
  ttl: CACHE_TTL_MS,
})

function getCached(url: string): CacheEntry | undefined {
  return cache.get(url)
}

function setCache(url: string, entry: CacheEntry, size: number) {
  cache.set(url, entry, { size: Math.max(1, size) })
}
```

**步骤6: 重定向处理**

```typescript
const MAX_REDIRECTS = 10

async function fetchWithRedirects(
  url: string,
  signal: AbortSignal,
  depth = 0,
): Promise<Response | RedirectInfo> {
  if (depth > MAX_REDIRECTS) {
    throw new Error(`Too many redirects (exceeded ${MAX_REDIRECTS})`)
  }

  const response = await fetchUrl(url, signal)
  
  if ([301, 302, 307, 308].includes(response.status)) {
    const location = response.headers.location
    const redirectUrl = new URL(location, url).toString()
    
    if (isSafeRedirect(url, redirectUrl)) {
      return fetchWithRedirects(redirectUrl, signal, depth + 1)
    }
    
    return { type: 'redirect', originalUrl: url, redirectUrl, statusCode: response.status }
  }
  
  return response
}

function isSafeRedirect(original: string, redirect: string): boolean {
  const o = new URL(original)
  const r = new URL(redirect)
  
  // 只允许同域重定向（含www变体）
  const stripWww = (h: string) => h.replace(/^www\./, '')
  return stripWww(o.hostname) === stripWww(r.hostname)
}
```

### 7.2 完整工具注册

```typescript
export const WebFetchTool = buildTool({
  name: 'WebFetch',
  searchHint: 'fetch and extract content from a URL',
  maxResultSizeChars: 100_000,
  shouldDefer: true,
  
  async description(input) {
    const { url } = input as { url: string }
    return `Claude wants to fetch content from ${new URL(url).hostname}`
  },
  
  userFacingName() {
    return 'Fetch'
  },
  
  get inputSchema() { return inputSchema },
  get outputSchema() { return outputSchema },
  
  isConcurrencySafe() { return true },
  isReadOnly() { return true },
  
  async checkPermissions(input, context) {
    // 权限检查逻辑
  },
  
  async call({ url, prompt }, context) {
    const start = Date.now()
    const response = await getURLMarkdownContent(url, context.abortController)
    
    if ('type' in response && response.type === 'redirect') {
      return formatRedirectResponse(response, prompt, start)
    }
    
    const result = await processContent(prompt, response.content, context.abortController.signal)
    
    return {
      data: {
        bytes: response.bytes,
        code: response.code,
        codeText: response.codeText,
        result,
        durationMs: Date.now() - start,
        url,
      }
    }
  },
})
```

### 7.3 预批准域名实现

```typescript
const PREAPPROVED_HOSTS = new Set([
  'docs.python.org',
  'react.dev',
  'nodejs.org',
  // ... 更多
])

// 分离主机名和路径前缀
const { HOSTNAME_ONLY, PATH_PREFIXES } = (() => {
  const hosts = new Set<string>()
  const paths = new Map<string, string[]>()
  
  for (const entry of PREAPPROVED_HOSTS) {
    const slash = entry.indexOf('/')
    if (slash === -1) {
      hosts.add(entry)
    } else {
      const host = entry.slice(0, slash)
      const path = entry.slice(slash)
      const prefixes = paths.get(host)
      if (prefixes) prefixes.push(path)
      else paths.set(host, [path])
    }
  }
  
  return { HOSTNAME_ONLY: hosts, PATH_PREFIXES: paths }
})()

export function isPreapprovedHost(hostname: string, pathname: string): boolean {
  if (HOSTNAME_ONLY.has(hostname)) return true
  
  const prefixes = PATH_PREFIXES.get(hostname)
  if (prefixes) {
    for (const p of prefixes) {
      if (pathname === p || pathname.startsWith(p + '/')) return true
    }
  }
  
  return false
}
```

---

## 8. 关键技术点总结

| 特性 | 实现方式 | 文件位置 |
|------|----------|----------|
| HTTP请求 | axios, 60s超时, 10MB限制 | `utils.ts:272-282` |
| HTML转Markdown | turndown库, 延迟加载 | `utils.ts:85-97` |
| 缓存 | LRU缓存, 15分钟TTL, 50MB限制 | `utils.ts:61-78` |
| 重定向 | 手动处理, 最多10次, 同域安全 | `utils.ts:262-329` |
| 安全检查 | URL验证 + 域名黑名单 + 预批准列表 | `utils.ts:139-203`, `preapproved.ts` |
| 内容处理 | Haiku模型, 100K字符限制 | `utils.ts:484-530` |
| 权限控制 | 预批准→拒绝→询问→允许→默认询问 | `WebFetchTool.ts:104-180` |

---

*文档生成时间: 2026-04-02*