# Coding Conventions

**Analysis Date:** 2026-04-02

## Naming Patterns

**Files:**
- TypeScript files: `.ts` extension for logic, `.tsx` for React components
- All imports use `.js` extension (ESM-style module resolution): `import { x } from './module.js'`
- Descriptive names: `useCanUseTool.tsx`, `sessionStorage.ts`, `permissionResult.ts`
- Directory-based organization: features grouped in directories (commands/, hooks/, utils/, types/)

**Functions:**
- camelCase for regular functions: `getLogDisplayTitle()`, `isAbortError()`, `hasPermissionsToUseTool()`
- Prefix convention for factory/creator functions: `createPermissionContext()`, `createStore()`
- Type guard functions prefixed with `is`: `isSyncHookJSONOutput()`, `isENOENT()`, `isFsInaccessible()`
- Hook functions prefixed with `use`: `useCanUseTool()`, `useReplBridge()`, `useVoiceIntegration()`

**Variables:**
- camelCase for local variables
- UPPER_CASE for constants: `MAX_IN_MEMORY_ERRORS`, `TICK_TAG`, `BASH_TOOL_NAME`
- Private/internal variables with underscore prefix in some contexts: `_temp`

**Types:**
- PascalCase for type/interface names: `PermissionResult`, `HookCallback`, `AppState`
- Branded types for IDs: `SessionId`, `AgentId` (string with readonly brand)
- Union types for discriminated results: `PermissionDecision = PermissionAllowDecision | PermissionAskDecision | PermissionDenyDecision`
- Generic type parameters: `Input extends Record<string, unknown>`, `Args extends unknown[]`

## Code Style

**Formatting:**
- Biome linter/formatter (primary tool)
- Source maps embedded in compiled output (base64 JSON)
- React Compiler runtime optimization pattern: `import { c as _c } from "react/compiler-runtime"`

**Linting:**
- Biome rules with inline ignores: `// biome-ignore lint/suspicious/noConsole: intentional console output`
- ESLint for specific rules: `// eslint-disable-next-line @typescript-eslint/no-require-imports`
- Custom ESLint rules detected:
  - `custom-rules/bootstrap-isolation` - enforces bootstrap module boundaries
  - `custom-rules/no-process-exit` - prevents direct process.exit() calls
  - `custom-rules/no-cross-platform-process-issues` - cross-platform safety
  - `custom-rules/require-tool-match-name` - tool name matching
  - `eslint-plugin-n/no-unsupported-features/node-builtins` - Node.js version compatibility

## Import Organization

**Order:**
1. React/compiler-runtime imports first
2. External packages (bun:bundle, lodash-es, zod)
3. SDK imports (@anthropic-ai/sdk)
4. Internal imports with relative paths
5. Type-only imports (`import type { ... }`)

**Import Markers:**
- ANT-ONLY markers must not be reordered: `// biome-ignore-all assist/source/organizeImports: ANT-ONLY import markers must not be reordered`
- Found in: `commands.ts:1`, `query.ts:1`, `cli/print.ts:1`

**Path Aliases:**
- `src/` alias used for deep imports: `import type { HookEvent } from 'src/entrypoints/agentSdkTypes.js'`
- Relative imports with `.js` extension mandatory: `import { x } from '../utils/errors.js'`

**Conditional/Lazy Imports:**
```typescript
// Dead code elimination pattern
/* eslint-disable @typescript-eslint/no-require-imports */
const agentsPlatform =
  process.env.USER_TYPE === 'ant'
    ? require('./commands/agents-platform/index.js').default
    : null
/* eslint-enable @typescript-eslint/no-require-imports */
```
- Used for feature-flagged modules
- Pattern: `feature('FLAG') ? require('./path.js').default : null`

## Error Handling

**Custom Error Classes:**
Location: `utils/errors.ts`
```typescript
export class ClaudeError extends Error {
  constructor(message: string) {
    super(message)
    this.name = this.constructor.name
  }
}

export class AbortError extends Error { ... }
export class ShellError extends Error { ... }
export class ConfigParseError extends Error { ... }
export class TeleportOperationError extends Error { ... }
```

**Telemetry-Safe Errors:**
```typescript
export class TelemetrySafeError_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS extends Error {
  readonly telemetryMessage: string
  constructor(message: string, telemetryMessage?: string) { ... }
}
```
- Long naming convention forces explicit verification
- Separate messages for user logs vs telemetry

**Type Guards for Errors:**
```typescript
export function isAbortError(e: unknown): boolean {
  return (
    e instanceof AbortError ||
    e instanceof APIUserAbortError ||
    (e instanceof Error && e.name === 'AbortError')
  )
}

export function isENOENT(e: unknown): boolean { ... }
export function isFsInaccessible(e: unknown): e is NodeJS.ErrnoException { ... }
export function getErrnoCode(e: unknown): string | undefined { ... }
```

**Error Helper Functions:**
```typescript
export function toError(e: unknown): Error { ... }
export function errorMessage(e: unknown): string { ... }
export function shortErrorStack(e: unknown, maxFrames = 5): string { ... }
export function classifyAxiosError(e: unknown): { kind: AxiosErrorKind; ... } { ... }
```

## Logging

**Framework:** Custom logging system

Location: `utils/log.ts`
- Error log sink interface: `ErrorLogSink`
- Queued error events before sink attachment
- MCP-specific logging: `logMCPError()`, `logMCPDebug()`
- In-memory error log with max 100 entries

**Debug Logging:**
```typescript
import { logForDebugging } from '../utils/debug.js'
logForDebugging("Disabling bypass permissions mode on mount")
```

**Patterns:**
- Console output requires Biome ignore: `// biome-ignore lint/suspicious/noConsole: intentional console output`
- Error logging via `logError()` from `utils/log.js`
- Analytics events via `logEvent()` from `services/analytics/`

## Comments

**When to Comment:**
- JSDoc for exported public APIs with `@param`, `@returns`, `@example`
- Inline comments for complex logic explanations
- TODO/FIXME comments with context identifiers: `// TODO(onKeyDown-migration): remove once REPL passes handleKeyDown`

**JSDoc/TSDoc:**
Location examples:
- `context/overlayContext.tsx:27-133`: Full JSDoc with `@param`, `@returns`, `@example`
- `utils/memoize.ts:29-39`: Function documentation with parameter descriptions
- `entrypoints/agentSdkTypes.ts:153-266`: Comprehensive API documentation

**TODO Comment Pattern:**
```typescript
// TODO(identifier): description
// TODO(onKeyDown-migration): remove once REPL passes handleKeyDown
// TODO(#23985): fold ExitPlanModeScanner into this poller
// TODO(prod-hardening): OAuth token may go stale over the 30min poll
```

## Function Design

**Size:** Large functions exist (main.tsx is 808KB) - likely generated/compiled output

**Parameters:**
- Object destructuring for complex parameters: `function App({ getFpsMetrics, stats, initialState, children }: Props)`
- Optional parameters with `?:` syntax
- Generic constraints: `Input extends { [key: string]: unknown }`

**Return Values:**
- Discriminated union returns: `{ behavior: 'allow' | 'deny' | 'ask' | 'passthrough' }`
- Async functions return `Promise<T>`
- Type guards return boolean with type predicate: `function isFsInaccessible(e: unknown): e is NodeJS.ErrnoException`

**Branded Types:**
Location: `types/ids.ts`
```typescript
export type SessionId = string & { readonly __brand: 'SessionId' }
export type AgentId = string & { readonly __brand: 'AgentId' }
export function asSessionId(id: string): SessionId { ... }
export function toAgentId(s: string): AgentId | null { ... }
```

## Module Design

**Exports:**
- Named exports preferred over default exports
- Type exports separate: `export type { ... }` vs `export { ... }`
- Re-export pattern: `export * from './submodule.js'`

**Barrel Files:**
Limited use - found in:
- `components/CustomSelect/index.ts`
- `entrypoints/sdk/coreTypes.ts`
- `entrypoints/agentSdkTypes.ts`

**Lazy Schema Pattern:**
Location: `utils/lazySchema.ts`
```typescript
export function lazySchema<T>(factory: () => T): () => T {
  let cached: T | undefined
  return () => (cached ??= factory())
}
```
- Defers Zod schema construction from module init to first access
- Used extensively in `types/hooks.ts` and `schemas/hooks.ts`

## React/Ink Component Patterns

**Component Structure:**
Location: `components/App.tsx`
```typescript
import { c as _c } from "react/compiler-runtime";
import React from 'react';

export function App(t0) {
  const $ = _c(9);  // React Compiler memoization cache
  const { getFpsMetrics, stats, initialState, children } = t0;
  // Conditional rendering with cache checks
  ...
}
```

**Hooks:**
Location: `hooks/` directory
- Heavy use of custom hooks (100+ hook files)
- Complex hooks: `useTypeahead.tsx` (214KB), `useVoiceIntegration.tsx` (100KB), `useReplBridge.tsx` (116KB)
- Permission hooks: `useCanUseTool.tsx`, hooks in `hooks/toolPermission/`

**Context Providers:**
Location: `state/AppState.tsx`, `context/` directory
- AppState pattern with store: `createStore()`, `AppStateProvider`
- Multiple context layers: `StatsProvider`, `FpsMetricsProvider`, `MailboxProvider`, `VoiceProvider`
- Overlay context for UI layers: `useRegisterOverlay()`

**Type-Only Component Props:**
```typescript
type Props = {
  getFpsMetrics: () => FpsMetrics | undefined;
  stats?: StatsStore;
  initialState: AppState;
  children: React.ReactNode;
}
```

## Zod Schema Patterns

**Schema Definition:**
Location: `schemas/hooks.ts`, `types/hooks.ts`, `bridge/*.ts`
```typescript
const BashCommandHookSchema = z.object({
  matcher: z.string().optional(),
  hooks: z.array(z.object({
    type: z.literal('bash'),
    command: z.string().describe('Shell command to execute'),
    timeout: z.number().optional(),
  })),
})
```

**Lazy Schema Usage:**
```typescript
export const syncHookResponseSchema = lazySchema(() =>
  z.object({
    continue: z.boolean().optional(),
    suppressOutput: z.boolean().optional(),
    ...
  })
)
```

**Type Inference:**
```typescript
export type PromptRequest = z.infer<ReturnType<typeof promptRequestSchema>>
```

## Feature Flags

**Pattern:**
```typescript
import { feature } from 'bun:bundle'

const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null

// Conditional logic
if (feature('TRANSCRIPT_CLASSIFIER') && result.decisionReason?.type === 'classifier') {
  setYoloClassifierApproval(toolUseID, result.decisionReason.reason)
}
```

**Detected Flags:**
- `VOICE_MODE`
- `TRANSCRIPT_CLASSIFIER`
- `BASH_CLASSIFIER`
- `BRIDGE_MODE`
- `DAEMON`
- `KAIROS`
- `KAIROS_BRIEF`
- `ULTRAPLAN`
- `TORCH`
- `UDS_INBOX`
- `FORK_SUBAGENT`
- `BUDDY`
- `WORKFLOW_SCRIPTS`
- `CCR_REMOTE_SETUP`
- `EXPERIMENTAL_SKILL_SEARCH`
- `KAIROS_GITHUB_WEBHOOKS`
- `HISTORY_SNIP`
- `PROACTIVE`

---

*Convention analysis: 2026-04-02*