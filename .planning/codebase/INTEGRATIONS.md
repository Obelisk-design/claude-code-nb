# External Integrations

**Analysis Date:** 2026-04-02

## APIs & External Services

**Anthropic Claude API:**
- Primary AI inference service for all Claude Code operations
- SDK: `@anthropic-ai/sdk` (see `services/api/client.ts`)
- Auth: OAuth 2.0 tokens or direct API key
  - OAuth: `Authorization: Bearer {token}`
  - API key: `x-api-key: {key}`
- Endpoint: Default `https://api.anthropic.com`, configurable via `ANTHROPIC_BASE_URL`
- Beta headers negotiated: multiple extension betas (see `constants/betas.ts`)
  - Extended thinking, 1M context, structured outputs, web search, prompt caching, fast mode

**AWS Bedrock:**
- Alternative inference provider hosted on AWS
- SDK: `@aws-sdk/client-sts`, `@aws-sdk/credential-providers`
- Auth: AWS credential chain (environment, ~/.aws/credentials, IAM roles)
- Configuration: `CLAUDE_CODE_USE_BEDROCK=1`, region via `AWS_REGION`/`AWS_DEFAULT_REGION`

**Google Vertex AI:**
- Alternative inference provider hosted on GCP
- SDK: `google-auth-library`
- Auth: GCP application default credentials
- Configuration: `CLAUDE_CODE_USE_VERTEX=1`, project via `ANTHROPIC_VERTEX_PROJECT_ID`

**Azure Foundry:**
- Alternative inference provider hosted on Azure
- Auth: API key or Azure AD `DefaultAzureCredential`
- Configuration: `CLAUDE_CODE_USE_FOUNDRY=1`, resource via `ANTHROPIC_FOUNDRY_RESOURCE` or `ANTHROPIC_FOUNDRY_BASE_URL`

**Claude.ai API Services:**
- Bootstrap configuration: `/api/claude_code/bootstrap`
- Settings sync: `/api/claude_code/settings_sync`
- Team memory sync: `/api/claude_code/team_memory`
- Fast/Penguin mode: `/api/claude_code_penguin_mode`
- MCP proxy: `https://mcp-proxy.anthropic.com/v1/mcp/{server_id}`

## Data Storage

**Databases:**
- No local database engine
- All persistent storage is file-based on local filesystem
- Memory consolidation uses markdown files in `~/.claude/memory/`

**File Storage:**
- Local filesystem only:
  - Settings: `~/.claude/settings.json`
  - Memory: `~/.claude/memory/` (CLAUDE.md topic files)
  - Team memory: `~/.claude/team-mem/`
  - Session history: JSON log files
  - Plugins: Cached in plugin directories with versioning
  - Debug dumps: `~/.config/claude/dump-prompts/` (internal builds only)

**Caching:**
- In-memory caches: `memoize`, `memoizeWithLRU` utilities
- Feature flags: Aggressively cached stale-while-revalidate pattern
- Settings cache: `utils/settings/settingsCache.ts` avoids repeated filesystem reads
- Plugin metadata cache: `utils/plugins/cacheUtils.ts`

## Authentication & Identity

**Anthropic OAuth:**
- PKCE flow with S256 code challenge (see `services/oauth/client.ts`)
- Authorize URL: `https://claude.com/cai/oauth/authorize`
- Token URL: `https://platform.claude.com/v1/oauth/token`
- Scopes: `user:inference`, `user:profile`, `user:sessions:claude_code`, `user:mcp_servers`, `user:file_upload`
- Local redirect: Dynamic port localhost callback
- Manual redirect fallback for headless environments
- Token refresh with lockfile to prevent concurrent refreshes

**Secure Storage:**
- macOS: Keychain for secure token/API key storage (see `utils/secureStorage/macOsKeychainHelpers.ts`)
- Linux: libsecret keyring integration
- Windows: Credential Manager (implied by cross-platform design)
- Fallback: File storage with restrictive permissions

**Custom OAuth:**
- Supports custom OAuth URLs via `CLAUDE_CODE_CUSTOM_OAUTH_URL` for FedStart/PubSec
- Allowed internal domains: `claude.fedstart.com`, `claude-ai.staging.ant.dev`

## Model Context Protocol (MCP)

**MCP Client:**
- Official SDK: `@modelcontextprotocol/sdk` (see `services/mcp/client.ts`)
- Supported transports:
  - `StdioClientTransport` - Local subprocess servers
  - `SSEClientTransport` - Server-sent events over HTTP
  - `StreamableHTTPClientTransport` - HTTP streaming
  - WebSocket - Custom implementation for remote connections
- Integration: MCP tools exposed as native Claude Code tools
- Resource support: List and read resources from MCP servers
- Connection monitoring: `MonitorTool` watches server connectivity

**MCP Authentication:**
- OAuth 2.0 for external MCP servers (see `services/mcp/auth.ts`)
- Dynamic client registration support
- Client metadata: `https://claude.ai/oauth/claude-code-client-metadata`

**Claude.ai MCP Registry:**
- Fetch official MCP server configurations from Claude.ai
- Subscriber eligibility checks
- See `services/mcp/claudeai.ts`, `services/mcp/officialRegistry.ts`

## Plugin/Extension System Architecture

**Plugin Types:**
- **Built-in plugins**: `{name}@builtin` registered in `plugins/builtinPlugins.ts`
- **Marketplace plugins**: Installed from Claude Code plugin marketplace
- **Local plugins**: Loaded from filesystem path

**Plugin Components:**
- **Skills**: Custom slash commands and prompts
- **Hooks**: Lifecycle event handlers (pre-command, post-command, etc.)
- **MCP Servers**: External tool/resource providers

**Plugin Architecture:**
- Manifest-driven: `plugin.json` defines capabilities
- Versioned caching: Downloads and caches plugin versions
- Dependency resolution: Handles inter-plugin dependencies
- User configuration: Per-plugin settings UI
- See `services/plugins/` for complete implementation

**Discovery:**
- Marketplace browsing via `/plugin browse` command
- Pagination, search, filtering
- See `commands/plugin/` CLI handlers

## Internal Component Integration

**Tool System:**
- All tools implement common `Tool` interface (see `Tool.ts`)
- Central tool registry at `tools.ts`
- Dynamic tool filtering: feature flags, permissions, user settings
- JSON schema caching: `toolSchemaCache.ts` optimizes prompt delivery
- Permission system: `tools/permissions/` handles risk classification and user approval

**Command System:**
- All CLI commands registered in `commands.ts`
- Modular organization: one feature per directory in `commands/`
- Typed command parsing via `@commander-js/extra-typings`

**React Component Integration:**
- Terminal UI built with React + custom Ink renderer
- Context-based state: `context/` directory provides React contexts
- Themes: `components/design-system/` provides themed primitives
- See `ink/` for custom rendering engine

**Bridge Mode (Remote Integration):**
- JWT-authenticated remote session bridge (see `bridge/`)
- Connect local CLI to claude.ai remote sessions
- Work modes: single-session, worktree, same-dir
- Trusted device tokens for persistent authentication
- WebSocket, SSE, and hybrid transports for server→client events
- See `cli/transports/` for transport implementations

**Upstream Proxy Integration:**
- Container-aware proxy relay in CCR (see `upstreamproxy/`)
- Uses `prctl(PR_SET_DUMPABLE, 0)` for memory security
- Reads session token from `/run/ccr/session_token`
- Excludes Anthropic API, GitHub, npm, PyPI from proxy

## Language Server Protocol (LSP)

**LSP Integration:**
- LSP client implementation (see `services/lsp/LSPClient.ts`)
- Server manager handles lifecycle (see `services/lsp/LSPServerManager.ts`)
- Diagnostic registry collects diagnostics (see `services/lsp/LSPDiagnosticRegistry.ts`)
- Provides context: errors and warnings from project to Claude

## Voice/Audio Integration

**Voice Input:**
- Native audio capture via `audio-capture-napi` NAPI module
- Platform-specific implementations:
  - macOS: CoreAudio
  - Linux: ALSA/PulseAudio with SoX fallback
  - Windows: WASAPI
- Streaming speech-to-text to Anthropic API (see `services/voiceStreamSTT.ts`)
- Push-to-talk keybinding support via `keybindings/`

## Analytics & Observability

**Feature Flags:**
- GrowthBook for runtime feature gating (see `services/analytics/growthbook.ts`)
- Client keys: Production `sdk-zAZezfDKGoZuXXKe`, Internal `sdk-xRVcrliHIlrg4og4`
- Caching: `getFeatureValue_CACHED_MAY_BE_STALE` allows stale data for performance

**Logging:**
- Datadog for aggregated structured logging (see `services/analytics/datadog.ts`)
- First-party event logging to BigQuery (see `services/analytics/firstPartyEventLoggingExporter.ts`)
- Local debug logging: `utils/debug.ts` (`logForDebugging`)
- Diagnostics logging: `utils/diagLogs.ts` (`logForDiagnosticsNoPII`)
- OpenTelemetry for distributed tracing: `@opentelemetry/*` imports throughout

## Git & System Integration

**Git Integration:**
- Git repository detection, worktree support
- Branch naming, remote URL parsing
- See `utils/git.ts`
- GitHub App installation flow: `/install-github-app` command

**Shell Integration:**
- Bash/PowerShell execution via `BashTool`/`PowerShellTool`
- Sandbox mode available for restricted execution
- See `tools/BashTool/`

**Vim Integration:**
- Vim text object motion support for vim-powered input
- See `vim/` directory with motions, operators, transitions

## Cross-Component Integration Patterns

**Feature Gating:**
- Compile-time: `feature('FEATURE_NAME')` from `bun:bundle` → dead code elimination
- Runtime: `getFeatureValue_CACHED_MAY_BE_STALE()` from GrowthBook

**Hooks System:**
- Plugin hooks for lifecycle events (see `utils/hooks.ts`)
- Hook matching: pattern matching on event types
- Async execution with error isolation

**Signals:**
- Custom signal implementation for reactive state: `utils/signal.js`
- Used instead of React state for non-UI reactive state

## Environment Configuration

**Required environment variables:**

Direct API mode:
- `ANTHROPIC_API_KEY` - API key for direct Anthropic access

AWS Bedrock:
- `CLAUDE_CODE_USE_BEDROCK=1`
- AWS credentials via standard AWS config chain
- `AWS_REGION` / `AWS_DEFAULT_REGION` (optional but recommended)

Google Vertex AI:
- `CLAUDE_CODE_USE_VERTEX=1`
- `ANTHROPIC_VERTEX_PROJECT_ID` - GCP project ID
- GCP credentials via application default
- `CLOUD_ML_REGION` (optional)

Azure Foundry:
- `CLAUDE_CODE_USE_FOUNDRY=1`
- `ANTHROPIC_FOUNDRY_RESOURCE` or `ANTHROPIC_FOUNDRY_BASE_URL`
- `ANTHROPIC_FOUNDRY_API_KEY` or Azure AD auth

**Secrets location:**
- macOS Keychain (encrypted storage)
- System keyring (Linux/Windows)
- `~/.claude/settings.json` - User-editable configuration
- Environment variables - For CI/automation
- File descriptor passing - For credential injection

## Webhooks & Callbacks

**Incoming:**
- OAuth callback: `http://localhost:{port}/callback`
- MCP OAuth callback: Configured per-server
- Manual redirect: `https://platform.claude.com/oauth/code/callback`

**Outgoing:**
- Anthropic Claude API messages
- MCP server connections (stdio, SSE, HTTP, WebSocket)
- Settings sync API
- Team memory sync API
- GrowthBook feature flag evaluation
- Datadog log ingestion
- Event batching with periodic flushing

---

*Integration audit: 2026-04-02*
