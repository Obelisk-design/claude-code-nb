# Architecture

**Analysis Date:** 2026-04-02

## Pattern Overview

**Overall:** Layered Architecture with Plugin/Extension System

**Key Characteristics:**
- React-based terminal UI via custom Ink renderer
- Tool-based capability system with permission layering
- Multi-agent orchestration with coordinator/worker pattern
- Feature flag driven conditional compilation (bun:bundle feature())
- State machine for conversation lifecycle (QueryEngine)
- Plugin/Skill extension points for customization

## Layers

**Entry Layer:**
- Purpose: CLI argument parsing, initialization orchestration, mode selection
- Location: `main.tsx`, `entrypoints/cli.tsx`, `entrypoints/init.ts`
- Contains: Commander CLI setup, bootstrap initialization, feature flag checks
- Depends on: All downstream layers
- Used by: Node.js/Bun runtime entry point

**State Layer:**
- Purpose: Global mutable state for session, React state store
- Location: `bootstrap/state.ts`, `state/AppStateStore.ts`, `state/store.ts`
- Contains: Session ID, cost tracking, model settings, permission context, telemetry counters
- Depends on: Types definitions, telemetry services
- Used by: All layers (read/write global session state)

**Query Engine Layer:**
- Purpose: Conversation lifecycle management, message processing, API interaction
- Location: `QueryEngine.ts`, `query.ts`
- Contains: QueryEngine class, message accumulation, tool call orchestration, compaction logic
- Depends on: Tools, Commands, MCP clients, Context providers
- Used by: REPL, SDK mode, headless sessions

**UI Layer:**
- Purpose: Terminal rendering, user interaction, component tree
- Location: `ink/ink.tsx`, `screens/REPL.tsx`, `components/`
- Contains: Custom React reconciler for terminal, Ink component library, REPL screen
- Depends on: State layer, Query engine, Hooks
- Used by: Interactive mode entry point

**Tools Layer:**
- Purpose: Capability execution (file ops, shell, web, agents)
- Location: `tools/`, `Tool.ts`, `tools.ts`
- Contains: Individual tool implementations (BashTool, FileEditTool, AgentTool, etc.), tool registry
- Depends on: Permissions, Utilities, State
- Used by: Query engine during tool_use block execution

**Commands Layer:**
- Purpose: Slash command execution, skill invocation
- Location: `commands/`, `commands.ts`
- Contains: Individual command implementations (/commit, /review, /plan, etc.), command registry
- Depends on: Tools, Services, Context
- Used by: REPL via PromptInput, QueryEngine

**Services Layer:**
- Purpose: External integrations, API clients, analytics
- Location: `services/`
- Contains: MCP client (`services/mcp/client.ts`), analytics (`services/analytics/`), API layer (`services/api/`), LSP manager
- Depends on: External SDKs, State
- Used by: Tools, Commands, QueryEngine

**Context Layer:**
- Purpose: System/user context injection, CLAUDE.md processing
- Location: `context.ts`, `utils/claudemd.ts`, `utils/queryContext.ts`
- Contains: Git status injection, memory files, user context assembly
- Depends on: Git utilities, File system
- Used by: QueryEngine for system prompt construction

## Data Flow

**Conversation Turn Flow:**

1. User input received via PromptInput component (`components/PromptInput/`)
2. Input processed via `processUserInput()` (`utils/processUserInput/`)
3. Context assembled via `getUserContext()` and `getSystemContext()` (`context.ts`)
4. QueryEngine.submitMessage() initiates API call (`QueryEngine.ts`)
5. Streaming response processed, tool_use blocks detected
6. Tool execution via `Tools` registry (`tools.ts`)
7. Permission checks via `useCanUseTool` hook (`hooks/useCanUseTool.tsx`)
8. Tool results accumulated, loop continues until completion
9. Messages stored via `recordTranscript()` (`utils/sessionStorage.ts`)

**State Management:**
- Global singleton state in `bootstrap/state.ts` (mutable, session-scoped)
- React state via `Store<T>` pattern (`state/store.ts`) - pub/sub with setState
- AppState combines settings, tasks, MCP state, plugins (`state/AppStateStore.ts`)

**Multi-Agent Flow (Coordinator Mode):**

1. Coordinator mode enabled via `feature('COORDINATOR_MODE')` and env var
2. User prompt sent to coordinator model (main loop)
3. Coordinator dispatches work via AgentTool to workers
4. Workers execute in background, notify via `<task-notification>` XML
5. Results synthesized by coordinator for user response

## Key Abstractions

**Tool:**
- Purpose: Capability unit for model invocation
- Examples: `tools/BashTool/BashTool.tsx`, `tools/FileEditTool/FileEditTool.tsx`, `tools/AgentTool/AgentTool.tsx`
- Pattern: `buildTool()` factory from `Tool.ts`, creates ToolDef with inputSchema/outputSchema/call function
- Structure: Each tool exports a Tool instance with Zod schemas, prompt handler, and execution logic

**Command:**
- Purpose: Slash command for user/model invocation
- Examples: `commands/commit.ts`, `commands/review.ts`, commands in `commands/` subdirectories
- Pattern: Command interface with type ('prompt' | 'flow'), name, description, getPromptForCommand()
- Structure: Command registry assembled in `commands.ts`, filtered by mode/permissions

**Skill:**
- Purpose: Reusable prompt template with optional hooks
- Examples: `skills/bundled/verify.ts`, `skills/bundled/claudeApi.ts`
- Pattern: `registerBundledSkill()` from `skills/bundledSkills.ts`, or disk-based skills in `.claude/skills/`
- Structure: BundledSkillDefinition with name, description, getPromptForCommand, optional files

**MCP Server:**
- Purpose: External tool/resource provider via Model Context Protocol
- Examples: Configured in `.claude/settings.json` mcpServers section
- Pattern: Client connection via `@modelcontextprotocol/sdk`, managed by `services/mcp/client.ts`
- Structure: MCPServerConnection with name, tools, resources, commands

**Agent:**
- Purpose: Spawned worker for parallel task execution
- Examples: General-purpose agent (`tools/AgentTool/built-in/generalPurposeAgent.ts`), custom agents in `.claude/agents/`
- Pattern: AgentDefinition with prompt, model, allowedTools, spawned via AgentTool
- Structure: LocalAgentTask or RemoteAgentTask for execution lifecycle

## Entry Points

**main.tsx:**
- Location: `main.tsx`
- Triggers: Node.js/Bun runtime invocation
- Responsibilities: CLI parsing (Commander), feature flag checks, initialization sequence, mode selection (REPL/SDK/headless)
- Key sequence: startupProfiler → MDM prefetch → keychain prefetch → Commander setup → init()

**entrypoints/cli.tsx:**
- Location: `entrypoints/cli.tsx`
- Triggers: main.tsx after CLI parsing
- Responsibilities: Mode-specific initialization, trust dialogs, OAuth flows, REPL/SDK launch

**entrypoints/init.ts:**
- Location: `entrypoints/init.ts`
- Triggers: During initialization sequence
- Responsibilities: Telemetry setup, analytics initialization, GrowthBook feature flags

**screens/REPL.tsx:**
- Location: `screens/REPL.tsx`
- Triggers: Interactive mode launch via `launchRepl()` (`replLauncher.tsx`)
- Responsibilities: Main UI loop, message display, prompt handling, keybindings, permission dialogs

## Extension Points

**Skills System:**
- Location: `skills/`, `skills/bundledSkills.ts`, `skills/loadSkillsDir.ts`
- Pattern: Register via `registerBundledSkill()` for compiled skills, or place in `.claude/skills/` for disk-based
- Extension: Add new BundledSkillDefinition or create skill file with frontmatter

**Plugins:**
- Location: `plugins/`, `utils/plugins/`
- Pattern: Plugin directories registered via `--plugin-dir` or settings, loaded by `pluginLoader.ts`
- Extension: Create plugin directory with commands.json, skills/, hooks/

**MCP Servers:**
- Location: Configured in settings, managed by `services/mcp/client.ts`
- Pattern: Add mcpServers entry to `.claude/settings.json` or `.claude.json`
- Extension: Configure stdio/SSE/WebSocket transport, tools auto-registered

**Custom Agents:**
- Location: `.claude/agents/` directory, loaded by `tools/AgentTool/loadAgentsDir.ts`
- Pattern: AgentDefinition JSON/YAML file with prompt, model, allowedTools
- Extension: Create agent definition file, invoke via AgentTool

**Hooks:**
- Location: `utils/hooks/`, settings hooks configuration
- Pattern: Pre/post tool hooks, session hooks, compact hooks via HooksSettings
- Extension: Configure in settings hooks section, or via SDK `registerHook()`

**Feature Flags:**
- Location: `bun:bundle` feature() calls throughout codebase
- Pattern: Conditional compilation via Bun's bundler, eliminates code from external builds
- Extension: Add feature flag in bundler config, gate with `feature('FLAG_NAME')`

## Error Handling

**Strategy:** Layered error handling with typed error classes

**Patterns:**
- `utils/errors.ts`: Custom error types (AbortError, ShellError, TeleportOperationError)
- `errorMessage()` helper for safe error extraction
- Tool validation returns `ValidationResult` type (success/failure with message)
- Permission handling returns `PermissionResult` type (allow/deny/ask)
- API errors categorized via `categorizeRetryableAPIError()` (`services/api/errors.ts`)
- Graceful shutdown via `gracefulShutdown()` (`utils/gracefulShutdown.ts`)

**Retry Logic:**
- API retries for rate limits, network errors
- Exponential backoff for MCP connections
- Permission prompt retry on invalid input

## Cross-Cutting Concerns

**Logging:** 
- `utils/log.ts`: logError, logMCPError, logMCPDebug
- `services/internalLogging.ts`: Ant-only internal logging
- `utils/diagLogs.ts`: Diagnostic logging (logForDiagnosticsNoPII)
- In-memory error log for debugging (`bootstrap/state.ts` inMemoryErrorLog)

**Validation:**
- Zod schemas for all tool inputs/outputs (`z.infer<ReturnType<typeof schema>>`)
- Settings validation via `utils/settings/validation.ts`
- Permission rule validation in permission setup

**Authentication:**
- OAuth via `services/oauth/`, `utils/auth.ts`
- API key via environment or keychain
- MCP server auth via `services/mcp/auth.ts`
- XAA IDP login for enterprise (`services/mcp/xaaIdpLogin.ts`)

**Telemetry:**
- OpenTelemetry metrics via `services/analytics/`
- GrowthBook feature flags (`services/analytics/growthbook.ts`)
- Event logging via `logEvent()` (`services/analytics/index.ts`)
- Cost tracking via `cost-tracker.ts`

**Permissions:**
- Permission modes: default, plan, auto, bypass (`utils/permissions/PermissionMode.ts`)
- Tool permission rules: alwaysAllow, alwaysDeny, alwaysAsk
- Permission context via `ToolPermissionContext` (`Tool.ts`)
- Permission prompts in UI via `components/permissions/`

## Most Complex/Core Module Deep Dive

**QueryEngine.ts** (~48KB, ~2000 lines estimated)

The QueryEngine is the central orchestrator for conversation lifecycle:

```typescript
// Core structure (simplified)
export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]
  private abortController: AbortController
  private permissionDenials: SDKPermissionDenial[]
  private totalUsage: NonNullableUsage
  private readFileState: FileStateCache

  constructor(config: QueryEngineConfig) { ... }
  
  async submitMessage(userMessage: Message): Promise<Message[]> { ... }
  
  private async processToolUseBlocks(...) { ... }
  
  private async handlePermissionCheck(...) { ... }
}
```

Key responsibilities:
- Message accumulation and storage
- API request construction with system prompt
- Streaming response handling
- Tool execution orchestration
- Permission denial tracking
- Usage/cost accumulation
- Compaction triggering
- Abort handling

The QueryEngine abstracts the core loop from both REPL and SDK/headless modes, enabling consistent behavior across interactive and non-interactive sessions.

---

*Architecture analysis: 2026-04-02*