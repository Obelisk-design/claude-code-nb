# Codebase Structure

**Analysis Date:** 2026-04-02

## Directory Layout

```
claude-code-nb/
├── main.tsx                 # Entry point (~785KB), CLI setup, initialization
├── bootstrap/               # Session bootstrap, global state
├── state/                   # React state store, AppState
├── ink/                     # Custom React terminal renderer (forked Ink)
├── screens/                 # Main UI screens (REPL, Resume, Doctor)
├── components/              # React UI components
├── commands/                # Slash command implementations (~100+)
├── tools/                   # Tool implementations (~40+)
├── skills/                  # Bundled skills, skill loading
├── plugins/                 # Plugin system, bundled plugins
├── services/                # External services, API clients, analytics
├── context/                 # React context providers
├── hooks/                   # React hooks for UI logic
├── utils/                   # Utility modules (permissions, settings, etc.)
├── types/                   # TypeScript type definitions
├── constants/               # Constants, prompts, product URLs
├── tasks/                   # Background task implementations
├── coordinator/             # Coordinator mode (multi-agent orchestration)
├── buddy/                   # Tamagotchi companion system
├── memdir/                  # Memory directory, memory consolidation
├── migrations/              # Settings migration scripts
├── entrypoints/             # Entry point modules (CLI, SDK, init)
├── remote/                  # Remote session management
├── server/                  # Direct connect server
├── voice/                   # Voice mode (optional feature)
├── assistant/               # Assistant mode (KAIROS feature)
├── proactive/               # Proactive suggestions (optional feature)
├── vim/                     # Vim mode keybindings
├── keybindings/             # Keybinding definitions
├── outputStyles/            # Output formatting styles
├── native-ts/               # Native TypeScript modules (Yoga layout)
├── moreright/               # "More right" feature
├── bridge/                  # Bridge mode for remote control
├── cli/                     # CLI handlers, transports
├── upstreamproxy/           # Upstream proxy configuration
├── public/                  # Public assets
├── schemas/                 # JSON schemas
├── QueryEngine.ts           # Conversation lifecycle engine
├── query.ts                 # Query function (wraps QueryEngine)
├── Tool.ts                  # Tool type definitions, buildTool factory
├── tools.ts                 # Tool registry, getTools()
├── Task.ts                  # Task definitions
├── tasks.ts                 # Task utilities
├── commands.ts              # Command registry, getCommands()
├── context.ts               # System/user context assembly
├── setup.ts                 # Setup logic
├── history.ts               # Input history management
├── cost-tracker.ts          # API cost tracking
├── ink.ts                   # Ink export
├── replLauncher.tsx         # REPL launch helper
├── dialogLaunchers.tsx      # Dialog launchers
├── interactiveHelpers.tsx   # Interactive mode helpers
├── projectOnboardingState.ts # Project onboarding state
```

## Directory Purposes

**bootstrap/:**
- Purpose: Session initialization, global mutable state
- Contains: `state.ts` (global state singleton), state getters/setters
- Key files: `bootstrap/state.ts` (session ID, cost, model settings, telemetry)

**state/:**
- Purpose: React state management, AppState store
- Contains: `store.ts` (Store<T> pub/sub pattern), `AppStateStore.ts` (AppState definition)
- Key files: `state/store.ts`, `state/AppStateStore.ts`, `state/AppState.ts`, `state/onChangeAppState.ts`

**ink/:**
- Purpose: Custom React terminal renderer (forked from Ink)
- Contains: React reconciler for terminal, screen management, input handling
- Key files: `ink/ink.tsx` (main Ink class), `ink/renderer.ts`, `ink/reconciler.ts`, `ink/screen.ts`, `ink/terminal.ts`

**screens/:**
- Purpose: Main UI screens for different modes
- Contains: REPL.tsx (main interactive loop), ResumeConversation.tsx, Doctor.tsx
- Key files: `screens/REPL.tsx` (~900KB, main UI), `screens/Doctor.tsx`, `screens/ResumeConversation.tsx`

**components/:**
- Purpose: React UI components for rendering
- Contains: 100+ component files for UI elements
- Key files: `components/App.tsx`, `components/PromptInput/`, `components/Messages.tsx`, `components/permissions/`

**commands/:**
- Purpose: Slash command implementations
- Contains: Individual command modules as directories or files
- Key files: `commands/commit.ts`, `commands/review.ts`, `commands/plan/index.ts`, `commands/config/index.ts`

**tools/:**
- Purpose: Tool implementations for model capabilities
- Contains: Individual tool directories with ToolName.tsx/Tool.tsx files
- Key files: `tools/BashTool/`, `tools/FileEditTool/`, `tools/FileReadTool/`, `tools/AgentTool/`, `tools/MCPTool/`

**skills/:**
- Purpose: Bundled skills system
- Contains: Skill registration, bundled skill definitions
- Key files: `skills/bundledSkills.ts`, `skills/loadSkillsDir.ts`, `skills/bundled/` (skill implementations)

**plugins/:**
- Purpose: Plugin system for extensions
- Contains: Bundled plugins, plugin loading
- Key files: `plugins/bundled/`, `plugins/builtinPlugins.ts`

**services/:**
- Purpose: External service integrations
- Contains: API clients, analytics, MCP, OAuth, LSP
- Key files: `services/mcp/client.ts` (MCP client), `services/analytics/`, `services/api/`, `services/lsp/`

**hooks/:**
- Purpose: React hooks for UI logic
- Contains: Custom hooks for state, permissions, tool usage
- Key files: `hooks/useCanUseTool.tsx`, `hooks/useAppState.ts`, `hooks/useMergedTools.ts`

**utils/:**
- Purpose: Utility modules for cross-cutting concerns
- Contains: Permissions, settings, git, file operations, model helpers
- Key files: `utils/permissions/`, `utils/settings/`, `utils/git/`, `utils/model/`, `utils/processUserInput/`

**types/:**
- Purpose: TypeScript type definitions
- Contains: Message types, permission types, plugin types, hooks types
- Key files: `types/message.ts`, `types/permissions.ts`, `types/hooks.ts`, `types/plugin.ts`, `types/ids.ts`

**constants/:**
- Purpose: Constants and prompts
- Contains: Product URLs, OAuth config, prompts, tool limits
- Key files: `constants/prompts.ts`, `constants/product.ts`, `constants/oauth.ts`, `constants/tools.ts`

**tasks/:**
- Purpose: Background task implementations
- Contains: LocalAgentTask, RemoteAgentTask, DreamTask, ShellTask
- Key files: `tasks/LocalAgentTask/`, `tasks/RemoteAgentTask/`, `tasks/DreamTask/`, `tasks/LocalShellTask/`

**coordinator/:**
- Purpose: Coordinator mode for multi-agent orchestration
- Contains: Coordinator system prompt, user context
- Key files: `coordinator/coordinatorMode.ts`

**buddy/:**
- Purpose: Tamagotchi companion system (BUDDY feature)
- Contains: Companion sprite, gacha system, notification handling
- Key files: `buddy/CompanionSprite.tsx`, `buddy/companion.ts`, `buddy/sprites.ts`

**memdir/:**
- Purpose: Memory directory for conversation memory
- Contains: Memory loading, age tracking, team memory
- Key files: `memdir/memdir.ts`, `memdir/memoryTypes.ts`, `memdir/paths.ts`

**entrypoints/:**
- Purpose: Entry point modules for different modes
- Contains: CLI entry, SDK entry, init logic
- Key files: `entrypoints/cli.tsx`, `entrypoints/init.ts`, `entrypoints/sdk/`

## Key File Locations

**Entry Points:**
- `main.tsx`: Primary entry, CLI setup
- `entrypoints/cli.tsx`: CLI mode entry
- `entrypoints/init.ts`: Initialization
- `screens/REPL.tsx`: Interactive REPL
- `replLauncher.tsx`: REPL launch helper

**Configuration:**
- `bootstrap/state.ts`: Global session state
- `state/AppStateStore.ts`: React state definition
- `utils/settings/settings.ts`: Settings loading
- `utils/config.ts`: Global config file

**Core Logic:**
- `QueryEngine.ts`: Conversation engine
- `query.ts`: Query wrapper
- `Tool.ts`: Tool definitions
- `tools.ts`: Tool registry
- `commands.ts`: Command registry
- `context.ts`: Context assembly

**Testing:**
- `tools/testing/`: Testing utilities
- No dedicated test directory detected in exploration (推测: tests may be co-located or external)

## Naming Conventions

**Files:**
- Tools: `tools/XTool/XTool.tsx` (e.g., `BashTool.tsx`, `FileEditTool.tsx`)
- Commands: `commands/X.ts` or `commands/X/index.ts`
- Skills: `skills/bundled/X.ts`
- Hooks: `hooks/useX.ts` or `hooks/useX.tsx`
- Components: `components/X.tsx`
- Utilities: `utils/X.ts` or `utils/X/X.ts`
- Types: `types/X.ts`

**Directories:**
- Tools: `tools/XTool/` (capitalized, suffixed with "Tool")
- Commands: `commands/X/` (lowercase)
- Services: `services/X/` (lowercase)
- Utils subdirs: `utils/X/` (lowercase)

**Exports:**
- Default export for main module (Tool, Command)
- Named exports for utilities, types
- Barrel files in some directories (index.ts)

## Module Dependency Graph (High-Level)

```
main.tsx
    ↓
entrypoints/cli.tsx → entrypoints/init.ts
    ↓
screens/REPL.tsx ←→ QueryEngine.ts
    ↓           ↓
hooks/       tools.ts ←→ Tool.ts
    ↓           ↓
state/       tools/XTool/
    ↓           ↓
bootstrap/state.ts ←→ services/ ←→ utils/
    ↓                       ↓
context.ts            services/mcp/
    ↓                       ↓
QueryEngine.ts        external SDKs (@anthropic-ai/sdk, @modelcontextprotocol/sdk)
```

**Key Dependency Relationships:**
- `main.tsx` imports from all major modules for initialization
- `QueryEngine.ts` depends on tools, commands, context, services
- `screens/REPL.tsx` depends on hooks, components, state, QueryEngine
- `Tool.ts` is imported by all tool implementations
- `bootstrap/state.ts` is imported pervasively for global state access
- `services/mcp/client.ts` depends on MCP SDK and provides tools to QueryEngine

## Core 5-10 Modules (Most Critical)

1. **`main.tsx`** (~785KB)
   - Justification: Entry point, CLI setup, orchestrates all initialization
   - Contains Commander setup, mode selection, feature flag checks

2. **`QueryEngine.ts`** (~48KB)
   - Justification: Core conversation lifecycle engine
   - Handles message accumulation, API calls, tool execution

3. **`screens/REPL.tsx`** (~900KB)
   - Justification: Main interactive UI, all user interaction
   - Contains message display, prompt handling, keybindings

4. **`ink/ink.tsx`** (~253KB)
   - Justification: Custom React terminal renderer
   - Provides React-to-terminal rendering, input handling

5. **`bootstrap/state.ts`** (~14KB)
   - Justification: Global session state singleton
   - Contains session ID, cost, model, telemetry state

6. **`Tool.ts`** (~30KB)
   - Justification: Tool type system and factory
   - Defines Tool interface, buildTool(), validation

7. **`services/mcp/client.ts`** (~122KB)
   - Justification: MCP server connection management
   - Provides external tools/resources from MCP servers

8. **`tools/AgentTool/AgentTool.tsx`** (~large)
   - Justification: Multi-agent orchestration tool
   - Spawns workers, handles async agent lifecycle

9. **`context.ts`** (~7KB)
   - Justification: System/user context injection
   - Assembles git status, CLAUDE.md, memory for prompts

10. **`tools/BashTool/BashTool.tsx`** (~large)
    - Justification: Shell command execution
    - Most frequently used tool, permission handling, sandbox

## Where to Add New Code

**New Feature:**
- Primary code: Create in appropriate layer (tools/, commands/, services/)
- UI components: `components/`
- Hooks: `hooks/`
- Tests: Co-located with implementation or separate test file

**New Tool:**
- Implementation: `tools/XTool/XTool.tsx`
- Register in: `tools.ts` (add to getTools())
- Permission rules: `utils/permissions/`
- UI rendering: `tools/XTool/UI.tsx` (tool result display)

**New Command:**
- Implementation: `commands/X.ts` or `commands/X/index.ts`
- Register in: `commands.ts` (import and add to command list)
- UI: Dialog component if needed

**New Component/Module:**
- Implementation: `components/X.tsx`
- Hooks: `hooks/useX.ts`
- Types: `types/X.ts`

**New Utility:**
- Shared helpers: `utils/X.ts` or `utils/X/`

**New Skill:**
- Bundled: `skills/bundled/X.ts`, register in `skills/bundledSkills.ts`
- Disk-based: `.claude/skills/X.md` (user project directory)

**New Plugin:**
- Create plugin directory with `commands.json`, `skills/`, `hooks/`
- Load via `--plugin-dir` or settings

## Special Directories

**`types/generated/`:**
- Purpose: Auto-generated types from protobuf/event schemas
- Generated: Yes (推测: from schema definitions)
- Committed: Yes

**`.planning/`:**
- Purpose: Planning documents (created by GSD tooling)
- Generated: Yes (by GSD commands)
- Committed: No (not in .gitignore but temporary)

**`native-ts/`:**
- Purpose: Native TypeScript modules for performance
- Contains: Yoga layout engine bindings
- Generated: No
- Committed: Yes

**`migrations/`:**
- Purpose: Settings migration scripts for backward compatibility
- Contains: Migration functions for settings updates
- Generated: No
- Committed: Yes

---

*Structure analysis: 2026-04-02*