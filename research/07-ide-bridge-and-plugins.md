# Chapter 7: IDE Bridge and Plugins

## 7.1 The Bridge System

The bridge system (`bridge/`) enables Claude Code to run as a background process that connects to IDEs — primarily VS Code and JetBrains. It's gated behind `feature('BRIDGE_MODE')` and consists of 31 TypeScript files.

### Architecture

```
┌──────────────────┐         ┌──────────────────┐
│   VS Code /      │  HTTP   │   Claude Code    │
│   JetBrains      │◄───────►│   Bridge Process │
│   Extension      │  Poll   │   (bridgeMain.ts)│
└──────────────────┘         └──────────────────┘
        │                            │
        │  File edits, diffs         │  Anthropic API
        │  Terminal output           │  Tool execution
        │  Permission dialogs        │  Session management
        ▼                            ▼
   IDE UI Layer                 LLM + Tools
```

### Bridge Components

```
bridge/
├── bridgeMain.ts              # Main bridge loop
├── bridgeApi.ts               # API client for bridge server
├── bridgeConfig.ts            # Bridge configuration
├── bridgeMessaging.ts         # Message handling
├── bridgePermissionCallbacks.ts # Permission UI delegation to IDE
├── bridgePointer.ts           # Bridge pointer management
├── bridgeStatusUtil.ts        # Status formatting
├── bridgeUI.ts                # Bridge-specific UI
├── bridgeDebug.ts             # Debug utilities
├── capacityWake.ts            # Capacity management
├── codeSessionApi.ts          # Code session API
├── createSession.ts           # Session creation
├── flushGate.ts               # Flush synchronization
├── inboundAttachments.ts      # Attachment handling
├── inboundMessages.ts         # Message processing
├── initReplBridge.ts          # REPL bridge initialization
├── jwtUtils.ts                # JWT token management
├── pollConfig.ts              # Polling configuration
├── pollConfigDefaults.ts      # Default poll intervals
├── remoteBridgeCore.ts        # Remote bridge functionality
├── replBridge.ts              # REPL-to-bridge adapter
├── replBridgeHandle.ts        # Bridge handle management
├── replBridgeTransport.ts     # Transport layer
├── sessionRunner.ts           # Session lifecycle
├── sessionIdCompat.ts         # Session ID compatibility
├── trustedDevice.ts           # Trusted device management
├── types.ts                   # Type definitions
└── workSecret.ts              # Work secret handling
```

### Bridge Protocol

The bridge uses a **polling-based protocol** rather than WebSockets:

```typescript
const getPollIntervalConfig = (): PollConfig => ({
  // Default poll intervals
  active: 100,    // ms when session is active
  idle: 1000,     // ms when session is idle
  background: 5000 // ms when no session
})
```

This design choice trades latency for reliability — polling works through proxies, firewalls, and network disruptions that might break WebSocket connections.

### Session Management

The bridge manages multiple sessions:

```typescript
const SPAWN_SESSIONS_DEFAULT = 32

async function isMultiSessionSpawnEnabled(): Promise<boolean> {
  return checkGate_CACHED_OR_BLOCKING('tengu_ccr_bridge_multi_session')
}
```

The multi-session spawn feature (gated behind `tengu_ccr_bridge_multi_session`) allows running up to 32 concurrent Claude Code sessions, each in their own context.

### Permission Delegation

When the bridge needs permission approval, it delegates to the IDE UI:

```typescript
// bridgePermissionCallbacks.ts
// Permissions are sent to the IDE extension which renders them in the IDE's native UI
// The user approves/denies in the IDE, and the response is polled back
```

This means VS Code users see permission dialogs rendered in VS Code's native UI rather than the terminal.

### JWT Authentication

Bridge sessions use JWT tokens for authentication:

```typescript
import { createTokenRefreshScheduler } from './jwtUtils.js'
```

Tokens are refreshed on a schedule to maintain long-lived bridge connections. The `trustedDevice.ts` module manages device trust, ensuring only authorized IDE installations can connect.

### Work Secrets

The `workSecret.ts` module handles secure session identification:

```typescript
export function buildSdkUrl(workSecret: string): string
export function buildCCRv2SdkUrl(workSecret: string): string
export function decodeWorkSecret(encoded: string): WorkSecret
export function registerWorker(workSecret: string): void
```

Work secrets are used to securely link bridge sessions to specific IDE instances, preventing unauthorized session hijacking.

## 7.2 Remote Bridge (CCR)

Claude Code Remote (CCR) extends the bridge to remote environments:

```typescript
const webCmd = feature('CCR_REMOTE_SETUP')
  ? require('./commands/remote-setup/index.js').default
  : null
```

CCR allows running Claude Code on a remote server while connecting from a local IDE. The `remoteBridgeCore.ts` handles the remote bridge lifecycle including:
- Remote session creation
- Worktree management for each session
- Log file management
- Graceful shutdown with `SIGTERM→SIGKILL` grace periods

```typescript
const DEFAULT_BACKOFF: BackoffConfig = {
  connInitialMs: 2_000,
  connCapMs: 120_000,      // 2 minutes
  connGiveUpMs: 600_000,   // 10 minutes
  generalInitialMs: 500,
  generalCapMs: 30_000,
  generalGiveUpMs: 600_000, // 10 minutes
  shutdownGraceMs: 30_000,  // 30 seconds
}
```

## 7.3 The Plugin System

The plugin system (`plugins/`) enables extending Claude Code with third-party functionality:

### Plugin Types

```typescript
type LoadedPlugin = {
  name: string
  manifest: { name: string; description: string; version: string }
  path: string
  source: string        // e.g., "name@marketplace" or "name@builtin"
  repository: string
  enabled: boolean
  isBuiltin: boolean
  hooksConfig?: HooksConfig
  mcpServers?: McpServerConfig[]
}
```

### Built-in Plugins

Built-in plugins ship with Claude Code and can be toggled by users:

```typescript
// plugins/builtinPlugins.ts
export function registerBuiltinPlugin(definition: BuiltinPluginDefinition): void
export function getBuiltinPlugins(): { enabled: LoadedPlugin[]; disabled: LoadedPlugin[] }
export function getBuiltinPluginSkillCommands(): Command[]
```

Built-in plugins use the `{name}@builtin` ID format. They can provide:
- **Skills** — Custom commands/capabilities
- **Hooks** — Pre/post tool use hooks
- **MCP servers** — Additional MCP server configurations

```typescript
type BuiltinPluginDefinition = {
  name: string
  description: string
  version: string
  defaultEnabled?: boolean  // Can be disabled by user
  isAvailable?: () => boolean  // Runtime availability check
  skills?: BundledSkillDefinition[]
  hooks?: HooksConfig
  mcpServers?: McpServerConfig[]
}
```

### External Plugins

External plugins are installed from marketplaces:

```typescript
import { VALID_INSTALLABLE_SCOPES, VALID_UPDATE_SCOPES }
  from './services/plugins/pluginCliCommands.js'
```

The `/plugin` command provides a UI for managing plugins:
- Install from marketplace
- Enable/disable
- Update
- Remove
- View plugin details

### Plugin Loading

Plugins are loaded via a cache-first strategy:

```typescript
const [skills, { enabled: enabledPlugins }] = await Promise.all([
  getSlashCommandToolSkills(getCwd()),
  loadAllPluginsCacheOnly(),
])
```

In headless/SDK mode, plugins are loaded from cache only (no network). The REPL can refresh via `/reload-plugins`. CCR environments pre-populate the cache via `CLAUDE_CODE_SYNC_PLUGIN_INSTALL` or `CLAUDE_CODE_PLUGIN_SEED_DIR`.

### Plugin Commands

Plugins can contribute commands (slash commands and skills):

```typescript
import {
  getPluginCommands,
  clearPluginCommandCache,
  getPluginSkills,
  clearPluginSkillsCache,
} from './utils/plugins/loadPluginCommands.js'
```

Plugin commands are merged with built-in commands during assembly, with built-in commands taking precedence on name conflicts.

### The Moved-To-Plugin Pattern

Some commands have been migrated to plugins:

```typescript
import createMovedToPluginCommand from './commands/createMovedToPluginCommand.js'
```

This pattern creates a stub command that tells users to install the plugin instead of running the original built-in command.

## 7.4 Hooks System

Hooks provide extension points throughout the tool lifecycle:

### Hook Types

The hooks system (defined in `utils/hooks.ts` and `types/hooks.ts`) supports:

1. **PreToolUse** — Run before a tool executes, can modify input or deny
2. **PostToolUse** — Run after a tool executes, can modify output
3. **PermissionRequest** — Run during permission checks, can auto-approve/deny
4. **PostSampling** — Run after model generates a response
5. **StopFailure** — Run when the model fails to stop (e.g., infinite loops)
6. **SessionStart** — Run at session initialization
7. **PreCompact** / **PostCompact** — Run around compaction

### Hook Sources

Hooks can come from:
- Skills (via `hooks` frontmatter in SKILL.md)
- Plugins (via `hooksConfig`)
- Built-in plugins
- Configuration files

### Hook Progress

Hooks can report progress to the UI:

```typescript
type HookProgress = {
  type: 'hook_progress'
  hookName: string
  message: string
}
```

## 7.5 MCP Integration

The Model Context Protocol (MCP) is deeply integrated:

### MCP Services

```
services/mcp/
├── client.ts      # MCP client management
├── types.ts       # MCP type definitions
└── officialRegistry.ts # Official MCP server registry
```

### MCP Tools

MCP servers provide tools that are merged into the tool pool:

```typescript
export function assembleToolPool(permissionContext, mcpTools): Tools {
  const builtInTools = getTools(permissionContext)
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)
  return uniqBy([...builtInTools, ...allowedMcpTools], 'name')
}
```

MCP tools carry `mcpInfo` metadata:
```typescript
mcpInfo?: { serverName: string; toolName: string }
```

### MCP Authentication

The `McpAuthTool` handles OAuth flows for MCP servers that require authentication. The `/mcp` command provides a UI for managing MCP server connections.

### MCP Resources

Beyond tools, MCP servers can provide resources:
- `ListMcpResourcesTool` — Browse available resources
- `ReadMcpResourceTool` — Read resource contents

### MCP Skills

MCP servers can also provide skills via `mcpSkillBuilders.ts`:

```typescript
export function registerMCPSkillBuilders(builders: {
  createSkillCommand: typeof createSkillCommand
  parseSkillFrontmatterFields: typeof parseSkillFrontmatterFields
}): void
```

This allows MCP servers to contribute custom slash commands that appear alongside built-in and user-defined skills.

---

*Next: [Chapter 8 — Memory, Skills, and Proactive Features](08-memory-skills-and-proactive-features.md)*
