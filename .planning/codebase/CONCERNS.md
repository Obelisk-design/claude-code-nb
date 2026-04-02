# Codebase Concerns

**Analysis Date:** 2026-04-02

## Security Concerns

### Credential Storage - Plaintext Fallback on Linux/Windows

- **Issue:** On non-macOS platforms (Linux, Windows), OAuth tokens and API keys are stored in plaintext JSON files due to lack of OS keychain integration
- **Files:** `utils/secureStorage/plainTextStorage.ts:19-84`, `utils/secureStorage/index.ts:9-17`
- **Impact:** High - Credentials stored at `~/.claude/.credentials.json` with 0o600 permissions can be read by any process running as the user. On shared systems or compromised machines, this is a credential exposure vector
- **Current Mitigation:** File permissions set to 0o600 (owner read/write only)
- **Fix Approach:** Implement libsecret (Linux) and Windows Credential Manager support. Note: TODO comment at `utils/secureStorage/index.ts:14` acknowledges this gap

### Keychain Command Injection via Username/Service Name

- **Issue:** macOS keychain operations use string interpolation in shell commands without proper escaping for `security` CLI
- **Files:** `utils/secureStorage/macOsKeychainStorage.ts:39-41`, `utils/secureStorage/macOsKeychainStorage.ts:118`
- **Impact:** Medium - If username or service name contains shell metacharacters, command injection is possible
- **Current Mitigation:** Hexadecimal encoding used for credential data (`-X` flag) but service name and username are still interpolated in quotes
- **Recommendation:** Use `execa` with array arguments (partially done in async path at lines 184-188) consistently for all security operations

### API Key Helper - Arbitrary Code Execution

- **Issue:** The `apiKeyHelper` setting allows executing arbitrary shell commands to retrieve API keys
- **Files:** `utils/auth.ts:469-499`, `utils/auth.ts:355-361`
- **Impact:** Critical - A malicious `apiKeyHelper` in project settings (`~/.claude/settings.json` or `.claude/settings.json`) could execute arbitrary code before trust dialog is shown
- **Current Mitigation:** `getConfiguredApiKeyHelper()` checks if helper is from project/local settings and requires trust acceptance (see `isApiKeyHelperFromProjectOrLocalSettings()`)
- **Fix Approach:** Document the security implications clearly. Consider restricting `apiKeyHelper` to user settings only (not project settings)

### JWT Signature Verification Bypass

- **Issue:** JWT token decoding for session ingress tokens does NOT verify signatures
- **Files:** `bridge/jwtUtils.ts:21-32`
- **Code:** `decodeJwtPayload()` explicitly parses JWT payload without signature verification
- **Impact:** Medium - A crafted token from a malicious source could inject arbitrary claims. However, tokens originate from trusted Anthropic infrastructure
- **Recommendation:** Document that this is intentional (trust in transport security) and add assertion comments. Consider signature verification for defense-in-depth

### JWT Credential Exposure in Command Line

- **Issue:** When using `--id-token <jwt>` option, the token appears in argv which can be captured in shell history and process listings
- **Files:** `commands/mcp/xaaIdpCommand.ts:162`
- **Impact:** Medium - Secrets leak to shell history and process listings when users manually input JWT tokens via command line
- **Current Mitigation:** Option primarily used for testing/conformance testing
- **Fix Approach:** Add a `--stdin` option to read JWT from stdin instead of argv as suggested in the TODO comment

### Session Ingress Token File Descriptor Handling

- **Issue:** Token read from file descriptors falls back to filesystem, potentially reading stale/malicious tokens
- **Files:** `utils/sessionIngressAuth.ts:18-86`, `utils/authFileDescriptor.ts`
- **Impact:** Medium - In CCR (containerized) environments, subprocesses may read tokens from predictable filesystem locations
- **Current Mitigation:** Token path can be overridden via `CLAUDE_SESSION_INGRESS_TOKEN_FILE` env var
- **Recommendation:** Validate token freshness and consider TTL-based rejection

### Sandbox Escape - dangerouslyDisableSandbox Parameter

- **Issue:** Tool execution allows `dangerouslyDisableSandbox` parameter to bypass OS-level sandboxing
- **Files:** `entrypoints/sandboxTypes.ts:113-120`, `services/tools/toolExecution.ts:1154-1155`
- **Impact:** High - If an attacker can influence tool inputs, they can escape the sandbox
- **Current Mitigation:** `allowUnsandboxedCommands` in managed settings can disable this capability organization-wide
- **Fix Approach:** Require explicit per-command user confirmation when `dangerouslyDisableSandbox` is used. Default `allowUnsandboxedCommands` to `false` in policy settings

### Bypass Permissions Mode (--dangerously-skip-permissions)

- **Issue:** CLI flag completely disables all permission checks including sandbox
- **Files:** `utils/permissions/permissionSetup.ts`, `components/BypassPermissionsModeDialog.tsx`
- **Impact:** Critical when misused - Allows unrestricted file access, command execution, and network access
- **Current Mitigation:** Trust dialog shown, analytics event logged (`tengu_bypass_permissions_mode_dialog_accept`)
- **Recommendation:** Consider requiring interactive confirmation at session start for this mode

### Unicode/Hidden Character Attack Surface

- **Issue:** Unicode sanitization has maximum iteration limit that could be exploited
- **Files:** `utils/sanitization.ts:25-65`
- **Code:** `MAX_ITERATIONS = 10` with crash on exhaustion
- **Impact:** Low - Denial of service possible via deeply nested unicode. Crash prevents malicious payload execution
- **Current Mitigation:** Crashes rather than processing potentially malicious content
- **Recommendation:** Add telemetry for iteration exhaustion events

---

## Authentication & Authorization Concerns

### OAuth Token Refresh Chain Failure

- **Issue:** Token refresh has limited retry attempts (MAX_REFRESH_FAILURES = 3) after which session loses auth
- **Files:** `bridge/jwtUtils.ts:57-58`, `bridge/jwtUtils.ts:186-206`
- **Impact:** Medium - Long-running sessions may lose authentication silently
- **Current Mitigation:** Retry delay and logging
- **Recommendation:** Add user notification when approaching failure limit

### OAuth Token Staleness in Long Polling

- **Issue:** Ultraplan remote polling can last up to 30 minutes. OAuth tokens may expire during the poll without refresh
- **Files:** `commands/ultraplan.tsx:20`
- **Impact:** Medium - Long-running ultraplan requests can fail mid-poll due to token expiration with no automatic recovery
- **Current Mitigation:** None - acknowledged as TODO with `TODO(prod-hardening)` comment
- **Fix Approach:** Implement token refresh during the polling cycle

### Environment Variable Credential Injection

- **Issue:** Multiple environment variables can inject credentials bypassing normal auth flows
- **Files:** `utils/auth.ts:111-206`
- **Variables:** `CLAUDE_CODE_OAUTH_TOKEN`, `CLAUDE_CODE_OAUTH_REFRESH_TOKEN`, `ANTHROPIC_API_KEY`, `ANTHROPIC_AUTH_TOKEN`
- **Impact:** Medium - Compromised environment variables (common in CI/CD) lead to credential exposure
- **Current Mitigation:** `isManagedOAuthContext()` prevents some fallbacks in CCR/Desktop contexts
- **Recommendation:** Add credential source logging and anomaly detection

### Organization UUID Header Exposure

- **Issue:** `X-Organization-Uuid` header sent with session key authentication
- **Files:** `utils/sessionIngressAuth.ts:117-131`
- **Impact:** Low - Organization UUID is not a secret but could be used for information gathering
- **Recommendation:** Acceptable for current architecture. Consider removing if moving to zero-trust model

---

## Data Handling & Privacy Concerns

### Telemetry Data Collection

- **Issue:** Extensive telemetry collected including session IDs, user IDs, organization IDs, email addresses
- **Files:** `services/analytics/metadata.ts:29-200`, `utils/telemetryAttributes.ts:29-71`
- **Data Points:** `user.id`, `session.id`, `user.email`, `organization.id`, `user.account_uuid`, `terminal.type`
- **Impact:** Medium - Privacy implications for enterprise users. Data sent to Datadog and BigQuery
- **Current Mitigation:** `isAnalyticsDisabled()` check, `_PROTO_*` key stripping before Datadog
- **Recommendation:** Document all telemetry points clearly. Provide single-flag opt-out

### MCP Tool Name Sanitization

- **Issue:** MCP tool names are sanitized to `mcp_tool` in analytics by default but can be logged in certain conditions
- **Files:** `services/analytics/metadata.ts:70-116`
- **Conditions:** `OTEL_LOG_TOOL_DETAILS=1`, `local-agent` entrypoint, official MCP URLs
- **Impact:** Low - MCP server names could reveal user infrastructure details
- **Current Mitigation:** Default sanitization, explicit gates for logging
- **Recommendation:** Maintain current approach - gate details logging explicitly

### Secret Scanner Coverage Gaps

- **Issue:** Client-side secret scanner only covers high-confidence patterns, may miss custom secret formats
- **Files:** `services/teamMemorySync/secretScanner.ts:39-224`
- **Impact:** Medium - Secrets not matching gitleaks patterns could be uploaded to team memory
- **Current Mitigation:** Curated rules from gitleaks with distinctive prefixes
- **Recommendation:** Document that scanner is a safety net, not a guarantee. Add custom pattern support

### Team Memory Sync - Data Exfiltration Vector

- **Issue:** Team memory sync uploads content to remote servers after secret scanning
- **Files:** `services/teamMemorySync/index.ts`, `services/teamMemorySync/secretScanner.ts`
- **Impact:** Medium - If secret scanner is bypassed, sensitive data could be exfiltrated
- **Current Mitigation:** Client-side secret scanning, PII-tagged fields for privileged access
- **Recommendation:** Add content preview before upload. Consider e2e encryption

---

## Code Injection & Execution Concerns

### Bash Command Parsing Complexity

- **Issue:** Complex bash AST parsing could have edge cases that bypass safety checks
- **Files:** `utils/bash/ast.ts`, `utils/bash/bashParser.ts`, `utils/bash/heredoc.ts`
- **Impact:** Medium - Malformed commands or parser bugs could allow dangerous patterns to slip through
- **Current Mitigation:** Dangerous pattern detection in `utils/permissions/dangerousPatterns.ts`
- **Recommendation:** Add parser fuzzing tests. Consider allow-list approach for sensitive operations

### Dangerous Pattern Detection Gaps

- **Issue:** Pattern-based dangerous command detection uses string matching which can be evaded
- **Files:** `utils/permissions/permissionSetup.ts:94-147`, `utils/permissions/dangerousPatterns.ts`
- **Impact:** Medium - Sophisticated attackers could craft commands that evade detection
- **Examples:** `python *` matches but `python3 -c '...'` might not in some configurations
- **Current Mitigation:** Multiple pattern types (prefix, wildcard, space)
- **Recommendation:** Add behavioral analysis or use sandboxing as primary defense

### PowerShell Command Execution

- **Issue:** PowerShell tool has similar dangerous pattern concerns with case-insensitive matching
- **Files:** `utils/permissions/permissionSetup.ts:157-200`
- **Impact:** Medium - PowerShell aliases (`iex`, `icm`) could bypass naive filters
- **Current Mitigation:** Comprehensive pattern list including aliases
- **Recommendation:** Consider disabling PowerShell tool by default in managed environments

---

## Undercover Mode Concerns

### Internal Information Leakage Prevention

- **Issue:** "Undercover Mode" prevents leaking internal Anthropic information but only activates for `USER_TYPE === 'ant'`
- **Files:** `utils/undercover.ts:1-90`
- **Purpose:** Strips internal model codenames, project names, attribution when contributing to public repos
- **Impact:** Positive security feature, no concerns - documented for awareness
- **Trigger:** `CLAUDE_CODE_UNDERCOVER=1` or auto-detection based on repo remote

### Undercover Auto-Detection Gap

- **Issue:** Undercover mode relies on `getRepoClassCached() !== 'internal'` check which may fail in edge cases
- **Files:** `utils/undercover.ts:34`
- **Impact:** Low - If internal repo check fails, stays undercover (safe default)
- **Recommendation:** Document edge cases where this could fail (e.g., new internal repos)

---

## Design Flaws & Architectural Issues

### main.tsx - God File Anti-Pattern

- **Issue:** Entry point file is ~800KB with massive import list and complex initialization (4683 lines)
- **Files:** `main.tsx`
- **Impact:** High - Maintainability nightmare, slow compilation, difficult to understand initialization order
- **Current Mitigation:** Lazy requires for some modules (`getTeammateUtils`, `assistantModule`)
- **Fix Approach:** Split into modular initialization stages. Move command definitions to separate files

### print.ts - Large Monolithic Printer Module

- **Issue:** 5594 lines in single file with mixed responsibilities
- **Files:** `cli/print.ts`
- **Impact:** High - High cognitive load, hard to navigate, frequent merge conflicts
- **Known Issues:** Mutable messages array comment acknowledges need cleanup (`cli/print.ts:1144`: TODO: Clean up mutable array)
- **Fix Approach:** Extract rendering logic, ask flow, and message processing into separate modules

### REPL.tsx - Large REPL Component

- **Issue:** 5005 lines in single React component
- **Files:** `screens/REPL.tsx`
- **Impact:** High - Hard to maintain, complex state interactions, slow incremental compiles
- **Fix Approach:** Split into smaller sub-components by responsibility (input, output, status, history)

### Debug Detection Before Imports Complete

- **Issue:** Debug detection at top of main.tsx calls `process.exit(1)` before graceful shutdown available
- **Files:** `main.tsx:266-271`
- **Code:** `"external" !== 'ant' && isBeingDebugged()` exits immediately
- **Impact:** Low - Intentional anti-debugging for external builds
- **Recommendation:** Add documentation for why this exists

### Circular Dependencies via Lazy Requires

- **Issue:** Multiple lazy `require()` calls used to avoid circular dependencies
- **Files:** `main.tsx:69-82`, `main.tsx:74-81`
- **Impact:** Medium - Runtime resolution of modules makes dependency graph opaque
- **Current Mitigation:** Type casts maintain type safety
- **Fix Approach:** Refactor to eliminate circular dependencies

### Stale-While-Error Cache Pattern

- **Issue:** Keychain cache serves stale data on errors, potentially masking credential issues
- **Files:** `utils/secureStorage/macOsKeychainStorage.ts:50-65`
- **Impact:** Medium - Could serve invalid credentials for extended periods
- **Current Mitigation:** Logging when serving stale data
- **Recommendation:** Add staleness TTL and force-refresh option

### Mutable Global State

- **Issue:** Mutable message array passed around and mutated during ask() flow
- **Files:** `cli/print.ts:1144-1145`
- **Impact:** Medium - Harder to reason about, potential unexpected side effects
- **Current Mitigation:** TODO comment acknowledges issue
- **Fix Approach:** Refactor to use immutable updates

### Global Top-Level Side Effects

- **Issue:** Main entrypoint has multiple top-level side effects for performance optimization with eslint explicitly disabled
- **Files:** `main.tsx:8-19`
- **Impact:** Medium - Makes module ordering hard to reason about, potential issues with testing
- **Fix Approach:** Encapsulate startup logic in an explicit initialization function

### Hacky React Context Workaround in MCP Command

- **Issue:** Global variable hack used to get React context outside component boundary
- **Files:** `commands/mcp/mcp.tsx:10`
- **Impact:** Low-Medium - Brittle, can lead to context mismatches with multiple server instances
- **Fix Approach:** Refactor to properly access context within component boundaries

### MCP Input Validation Missing

- **Issue:** MCP server entrypoint does not validate input argument types
- **Files:** `entrypoints/mcp.ts:136`
- **Impact:** Medium - Malformed inputs from clients could cause unexpected behavior or crashes
- **Current Mitigation:** None - TODO comment present
- **Fix Approach:** Add Zod schema validation at the entrypoint

---

## Performance Concerns

### Startup Time - Heavy Import Chain

- **Issue:** Startup imports ~200 modules synchronously before first user interaction
- **Files:** `main.tsx:1-210`, `utils/startupProfiler.ts`
- **Impact:** High - Slow startup on low-spec machines, affects perceived performance
- **Current Mitigation:** `profileCheckpoint` for profiling, parallel MDM/keychain prefetch
- **Fix Approach:** Implement lazy loading for non-critical features. Use dynamic imports

### Sync Filesystem Operations in Hot Paths

- **Issue:** Multiple synchronous filesystem operations block the event loop
- **Files:** `utils/secureStorage/macOsKeychainStorage.ts:39-46`, `utils/sandbox/sandbox-adapter.ts:273-274`
- **Code:** `execSyncWithDefaults_DEPRECATED`, `statSync` in sandbox adapter
- **Impact:** Medium - UI jank during credential reads, sandbox initialization
- **Current Mitigation:** Cache results where possible
- **Recommendation:** Migrate to async operations with loading states

### Token Estimation Performance

- **Issue:** Token counting uses heuristic estimation that may be slow for large files
- **Files:** `services/tokenEstimation.ts`
- **Impact:** Low - Affects context analysis for large codebases
- **Recommendation:** Consider caching per-file token counts

### Large File State Cache Compaction

- **Issue:** Full cache compaction runs when size limit hit
- **Files:** `utils/fileStateCache.ts`
- **Impact:** Low - Brief blocking when limit reached on large sessions
- **Recommendation:** Incremental or LRU-based compaction

---

## Maintainability Issues

### Feature Flag Proliferation

- **Issue:** Extensive use of `feature('FEATURE_NAME')` creates parallel code paths
- **Files:** Throughout codebase, e.g., `main.tsx:76`, `main.tsx:80`, `main.tsx:171`
- **Impact:** Medium - Difficult to test all code paths, dead code accumulation
- **Current Mitigation:** TypeScript ensures type safety
- **Recommendation:** Regular feature flag cleanup sprints

### Deprecated Function Usage

- **Issue:** Multiple `_DEPRECATED` functions still in use
- **Files:** `utils/auth.ts:58`, `utils/slowOperations.ts:9`
- **Examples:** `execSyncWithDefaults_DEPRECATED`, `writeFileSync_DEPRECATED`
- **Impact:** Low - Accumulating technical debt
- **Recommendation:** Track deprecation removal in roadmap

### TODO Accumulation

- **Issue:** Multiple TODOs indicate unfinished security/feature work
- **Files:**
  - `utils/secureStorage/index.ts:14` - add libsecret support for Linux (security)
  - `entrypoints/mcp.ts:136` - validate input types with zod (security)
  - `commands/mcp/xaaIdpCommand.ts:162` - read JWT from stdin to avoid shell history (security)
  - `commands/ultraplan.tsx:20` - OAuth token may go stale over 30min poll (reliability)
  - `cli/print.ts:1144` - clean up mutable array (maintainability)
  - `components/Settings/Config.tsx:263` - add MCP servers UI (feature completeness)
  - `components/ScrollKeybindingHandler.tsx:568` - implement search / , n/N (feature completeness)
- **Impact:** Medium - TODOs may represent security and functionality gaps
- **Recommendation:** Prioritize security-related TODOs for completion

### Incomplete Features

- **Issue:** Several features are partially implemented but incomplete
- **Files:**
  - `commands/plugin/BrowseMarketplace.tsx:682` - TODO: Actually scan local plugin directories
  - `components/ResumeTask.tsx:168` - TODO: include branch name when API returns it
  - `components/permissions/ExitPlanModePermissionRequest/ExitPlanModePermissionRequest.tsx:188` - TODO: Delete the branch after moving to V2
  - `components/permissions/PermissionRequest.tsx:145` - TODO: Move this to Tool.renderPermissionRequest
  - `cli/print.ts:2919` - TODO: use readonly types to avoid casting
  - `cli/print.ts:1876` - TODO(custom-tool-refactor): Should move to the init message, like browser

- **Impact:** Low-Medium - Some functionality incomplete, dead code from migrations remains
- **Recommendation:** Track incomplete features in backlog

---

## Hardcoded Values & Configuration Issues

### Hardcoded API Endpoints

- **Issue:** API endpoints hardcoded throughout codebase
- **Files:** `constants/oauth.ts:85-140`, `constants/product.ts:1-77`
- **Examples:** `https://api.anthropic.com`, `https://claude.ai`, `https://mcp-proxy.anthropic.com`
- **Impact:** Low - Cannot use alternative endpoints without code changes
- **Current Mitigation:** Staging URLs available, env var overrides for some paths
- **Recommendation:** Consolidate endpoint configuration

### Hardcoded Timeouts and Limits

- **Issue:** Multiple hardcoded timeout values without configuration
- **Files:** `utils/Shell.ts:44`, `bridge/jwtUtils.ts:52-61`
- **Values:** `DEFAULT_TIMEOUT = 30 * 60 * 1000` (30 min), `TOKEN_REFRESH_BUFFER_MS = 5 * 60 * 1000` (5 min)
- **Impact:** Low - Cannot tune for different environments
- **Recommendation:** Make configurable via settings

### Hardcoded Secret Patterns

- **Issue:** Secret scanner patterns are hardcoded, cannot be extended by users
- **Files:** `services/teamMemorySync/secretScanner.ts:48-224`
- **Impact:** Low - Cannot add organization-specific secret patterns
- **Recommendation:** Add configuration for custom patterns

---

## Permission System Concerns

### Permission Mode Transitions

- **Issue:** Complex state machine for permission modes (plan, auto, default) with multiple transition paths
- **Files:** `utils/permissions/permissionSetup.ts`, `bootstrap/state.ts`
- **Impact:** Medium - Edge cases in mode transitions could leave security gaps
- **Current Mitigation:** `handleAutoModeTransition`, `handlePlanModeTransition` handlers
- **Recommendation:** Add state machine diagram to documentation, comprehensive tests

### Trust Dialog State Persistence

- **Issue:** Trust dialog acceptance stored in config file which could be modified by malicious code
- **Files:** `utils/config.ts:110-112`, `utils/auth.ts:366-378`
- **Impact:** Medium - Once trust accepted, project settings can execute arbitrary code
- **Current Mitigation:** Per-project trust acceptance
- **Recommendation:** Add integrity check for trust acceptance records

---

## Sandbox Mechanism Concerns

### Bare Git Repo Attack Vector

- **Issue:** Code specifically handles "bare git repo attack" where attacker plants git metadata
- **Files:** `utils/sandbox/sandbox-adapter.ts:267-280`
- **Code:** Security comment explains git's `is_git_directory()` behavior
- **Impact:** Medium - Planting HEAD + objects/ + refs/ could escape sandbox
- **Current Mitigation:** `bareGitRepoScrubPaths` and `denyWrite` for these paths
- **Recommendation:** Good coverage, add tests for this attack vector

### Sandbox Filesystem Path Resolution Complexity

- **Issue:** Multiple path resolution functions with subtle differences
- **Files:** `utils/sandbox/sandbox-adapter.ts:99-146`
- **Functions:** `resolvePathPatternForSandbox`, `resolveSandboxFilesystemPath`
- **Impact:** Low - Maintenance burden, potential for inconsistency
- **Recommendation:** Consolidate into single well-documented function

### Network Isolation Weakening Option

- **Issue:** `enableWeakerNetworkIsolation` allows trustd.agent access, reducing security
- **Files:** `entrypoints/sandboxTypes.ts:126-133`
- **Impact:** Medium - Opens data exfiltration vector through trustd
- **Current Mitigation:** Default `false`, requires explicit opt-in
- **Recommendation:** Log analytics event when enabled, document risks

---

## Platform-Specific Gaps

### No Native Keychain on Linux

- **Issue:** No libsecret integration for Linux desktop. Falls back to plaintext.
- **Files:** `utils/secureStorage/index.ts:14-16`
- **Impact:** High - All credentials stored in plaintext on Linux
- **Fix Approach:** Implement libsecret support via `libsecret` npm package

### No Native Keychain on Windows

- **Issue:** No Windows Credential Manager integration. Falls back to plaintext.
- **Files:** `utils/secureStorage/index.ts:14-16`
- **Impact:** High - All credentials stored in plaintext on Windows
- **Fix Approach:** Implement Windows Credential Manager support via `wincred` or equivalent

---

## Test Coverage Gaps

### Security-Critical Code Paths

- **Issue:** No visible test files for security-critical modules in this analysis
- **Files:** `utils/secureStorage/`, `utils/permissions/`, `utils/sanitization.ts`
- **Impact:** High - Security bugs could go undetected
- **Recommendation:** Add unit tests for all security modules. Add integration tests for auth flows

### Platform-Specific Code

- **Issue:** Platform-specific keychain code (macOS, future Linux/Windows) difficult to test across all CI environments
- **Files:** `utils/secureStorage/macOsKeychainStorage.ts`, `utils/secureStorage/macOsKeychainHelpers.ts`
- **Impact:** Medium - Changes can break keychain functionality that goes undetected until release
- **Recommendation:** Add mock-based unit tests, integration tests on platform-specific CI runners

### Error Handling Edge Cases

- **Issue:** Many `catch (_e)` blocks silently swallow errors
- **Files:** `utils/secureStorage/macOsKeychainStorage.ts:47-49`, `utils/auth.ts:155-157`
- **Impact:** Medium - Debugging failures difficult, could mask security issues
- **Recommendation:** Add structured error logging with error types

### MCP Server Entry Point

- **Issue:** Standalone MCP server entrypoint likely lacks test coverage
- **Files:** `entrypoints/mcp.ts`
- **Impact:** Medium - Changes to input handling breaks MCP server without notice
- **Recommendation:** Add integration tests for MCP server

---

## Summary of Critical Concerns

| Concern | Severity | Exploitability | Priority |
|---------|----------|----------------|----------|
| Plaintext credential storage (Linux/Windows) | High | High | P0 |
| API Key Helper code execution | Critical | Medium | P0 |
| dangerouslyDisableSandbox parameter | High | Medium | P0 |
| JWT credential exposure via command line | Medium | High | P0 |
| Bypass Permissions mode | Critical | Low | P1 |
| Bash/PowerShell command parsing edge cases | Medium | Medium | P1 |
| Missing MCP input validation | Medium | Medium | P1 |
| OAuth token refresh failure handling | Medium | Low | P2 |
| OAuth token staleness in 30min polling | Medium | Low | P2 |
| Telemetry data collection | Medium | Low | P2 |
| Monolithic files (main.tsx, print.ts, REPL.tsx) | Medium | N/A | P2 |
| Test coverage gaps (security modules) | Medium | N/A | P2 |

---

*Concerns audit: 2026-04-02*