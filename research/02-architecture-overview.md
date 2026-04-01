# Chapter 2: Architecture Overview

## 2.1 Technology Stack

Claude Code is built on a surprisingly modern and opinionated technology stack:

| Layer | Technology | Notes |
|-------|-----------|-------|
| Runtime | **Bun** | Anthropic acquired Bun in late 2025; Claude Code uses `bun:bundle` for compile-time feature flags |
| Language | **TypeScript** | ~1,900 `.ts`/`.tsx` files |
| UI Framework | **React + Ink** | React for the terminal — fully reactive CLI |
| CLI Framework | **Commander.js** (`@commander-js/extra-typings`) | Type-safe CLI argument parsing |
| Schema Validation | **Zod v4** | Tool input schemas, config validation |
| API Client | **@anthropic-ai/sdk** | Official Anthropic SDK |
| MCP Support | **@modelcontextprotocol/sdk** | Model Context Protocol integration |
| Feature Flags | **GrowthBook** | Runtime feature gating (A/B testing) |
| Compile-time Flags | **`bun:bundle` `feature()`** | Dead code elimination for unreleased features |
| Build | **Bun bundler** | Single-file bundle with optional source maps |

### The Bun Connection

One of the most significant architectural choices is the deep integration with Bun. The `feature()` function from `bun:bundle` is used extensively for compile-time dead code elimination:

```typescript
import { feature } from 'bun:bundle'

const SleepTool =
  feature('PROACTIVE') || feature('KAIROS')
    ? require('./tools/SleepTool/SleepTool.js').SleepTool
    : null
```

This means entire subsystems (KAIROS, COORDINATOR_MODE, VOICE_MODE, etc.) are completely stripped from the production bundle when their feature flags are disabled. The internal Anthropic builds ("ant" builds) include tools and features not available to external users, controlled by `process.env.USER_TYPE === 'ant'`.

## 2.2 High-Level Architecture

The architecture follows a pipeline model:

```
User Input → CLI Parser → Query Engine → LLM API → Tool Execution Loop → Terminal UI
```

In more detail:

```
┌──────────────────────────────────────────────────────────────────┐
│                          main.tsx                                 │
│  (Commander.js CLI parser, 4,683 lines)                          │
│  - Parses CLI args (--model, --print, --resume, etc.)            │
│  - Initializes GrowthBook, analytics, auth                       │
│  - Routes to: REPL mode | Print mode | Bridge mode | SDK mode    │
└─────────────────────────┬────────────────────────────────────────┘
                          │
                ┌─────────▼──────────┐
                │   replLauncher.tsx  │
                │   QueryEngine.ts   │
                └─────────┬──────────┘
                          │
          ┌───────────────▼───────────────┐
          │          query.ts             │
          │  (Core LLM query loop)        │
          │  - Assembles system prompt    │
          │  - Calls Anthropic API        │
          │  - Processes tool calls       │
          │  - Handles streaming          │
          │  - Auto-compaction            │
          └───────────────┬───────────────┘
                          │
     ┌────────────────────▼────────────────────┐
     │        Tool Execution (tools.ts)         │
     │  BashTool, FileReadTool, FileEditTool,  │
     │  AgentTool, SkillTool, WebFetchTool...  │
     └────────────────────┬────────────────────┘
                          │
          ┌───────────────▼───────────────┐
          │    Permission System           │
          │  hooks/toolPermission/         │
          │  - Rule matching               │
          │  - Classifier auto-approval    │
          │  - Hook-based overrides        │
          │  - Interactive prompts         │
          └───────────────────────────────┘
```

## 2.3 Module Map

The `src/` directory is organized into these major modules:

### Core Engine

| Path | Purpose | Lines |
|------|---------|-------|
| `main.tsx` | CLI entrypoint, Commander.js setup, mode routing | 4,683 |
| `QueryEngine.ts` | Session/conversation lifecycle manager | 1,295 |
| `query.ts` | Core LLM query loop with streaming and tool execution | 1,729 |
| `Tool.ts` | Tool type definitions, permission types | 792 |
| `tools.ts` | Tool registry, assembly, filtering | 389 |
| `commands.ts` | Slash command registry | 754 |
| `context.ts` | System/user context assembly | ~200 |
| `cost-tracker.ts` | Token cost tracking | ~300 |

### Directories

| Directory | Purpose | Key Files |
|-----------|---------|-----------|
| `tools/` | 40+ tool implementations | AgentTool, BashTool, FileReadTool, etc. |
| `commands/` | ~100 slash command implementations | compact, commit, diff, memory, etc. |
| `components/` | ~146 React/Ink UI components | Messages, Permissions, Spinner, etc. |
| `hooks/` | React hooks + permission handlers | toolPermission, useCanUseTool, etc. |
| `services/` | External service integrations | api, analytics, mcp, oauth, compact, etc. |
| `bridge/` | IDE bridge (VS Code, JetBrains) | bridgeMain, bridgeApi, sessionRunner |
| `coordinator/` | Multi-agent coordinator mode | coordinatorMode.ts |
| `plugins/` | Plugin system | builtinPlugins, bundled/ |
| `skills/` | Skill system | loadSkillsDir, bundledSkills, etc. |
| `memdir/` | Memory directory system | memdir, memoryScan, memoryTypes |
| `state/` | Application state management | AppState |
| `types/` | Type definitions | message, permissions, hooks, etc. |
| `utils/` | ~330+ utility files | Everything from git to permissions to model |
| `tasks/` | Background task system | Task management for multi-agent |
| `voice/` | Voice mode | voiceModeEnabled |
| `assistant/` | KAIROS assistant mode | sessionHistory |
| `remote/` | Remote session management | Remote execution |
| `schemas/` | Schema definitions | Various schemas |
| `screens/` | Screen layouts | Terminal screen management |
| `server/` | Server components | API server endpoints |
| `ink/` | Custom Ink extensions | Terminal rendering optimizations |
| `vim/` | Vim mode support | Vim keybindings |

## 2.4 Execution Modes

Claude Code supports multiple execution modes, selected via CLI flags or environment:

### 2.4.1 Interactive REPL Mode (Default)

The primary mode. Launches a full React/Ink terminal UI with:
- Input buffer with typeahead
- Streaming response display
- Permission dialogs
- Tool progress indicators
- Search and scroll
- Vim mode support

Entry: `replLauncher.tsx` → `App.tsx` → hooks wire up `QueryEngine`

### 2.4.2 Print Mode (`--print`)

Non-interactive, single-shot mode. Sends one prompt and prints the response to stdout. Used for scripting and CI.

Entry: `main.tsx` → `QueryEngine` directly (no React/Ink UI)

### 2.4.3 SDK Mode

Programmatic API via the `@anthropic-ai/claude-code` npm package. Returns an async generator of typed messages.

Entry: `entrypoints/` → `QueryEngine.submitMessage()` yields `SDKMessage` events

### 2.4.4 Bridge Mode (IDE Integration)

Runs as a background process that connects to VS Code or JetBrains IDEs via a polling-based bridge protocol.

Entry: `bridge/bridgeMain.ts` — polls for incoming requests, spawns sessions

### 2.4.5 Assistant Mode (KAIROS — Internal Only)

A persistent, always-on assistant mode that:
- Maintains append-only daily logs
- Runs proactive background tasks
- Consolidates memory nightly via "dreaming"
- Sends push notifications

Entry: `assistant/` — gated behind `feature('KAIROS')`

## 2.5 The `main.tsx` Entrypoint

At 4,683 lines, `main.tsx` is by far the largest single file. It handles:

1. **Startup profiling** — timestamps for every phase of initialization
2. **MDM (Mobile Device Management) prefetch** — reads managed settings in parallel
3. **Keychain prefetch** — pre-fetches OAuth tokens from macOS keychain
4. **Commander.js CLI setup** — defines all CLI flags and options
5. **Authentication** — OAuth flow, API key validation, AWS/GCP credential checks
6. **GrowthBook initialization** — feature flag system
7. **Trust dialog** — first-run trust acceptance
8. **Mode routing** — dispatches to REPL, Print, Bridge, or SDK mode
9. **Plugin/Skill loading** — loads custom commands, skills, plugins
10. **MCP server initialization** — connects to configured MCP servers

The file opens with aggressive startup optimization:

```typescript
import { profileCheckpoint, profileReport } from './utils/startupProfiler.js';
profileCheckpoint('main_tsx_entry');
import { startMdmRawRead } from './utils/settings/mdm/rawRead.js';
startMdmRawRead();
import { ensureKeychainPrefetchCompleted, startKeychainPrefetch }
  from './utils/secureStorage/keychainPrefetch.js';
startKeychainPrefetch();
```

Side-effect imports fire MDM subprocess reads and keychain prefetches **before** the rest of the 135ms of imports even load. This is game-engine-style optimization applied to a CLI tool.

## 2.6 State Management

Application state is managed through a centralized `AppState` type (in `state/AppState.ts`) with a `getAppState`/`setAppState` pattern:

```typescript
getAppState(): AppState
setAppState(f: (prev: AppState) => AppState): void
```

This is essentially a minimal Redux-like store without the ceremony. The state includes:
- Tool permission context (rules, mode, working directories)
- MCP connection state
- Fast mode settings
- File history state
- Attribution state (for git commit attribution)
- Background task registry

For subagents, `createSubagentContext` creates a forked state that can selectively write back to the root store (e.g., for background task registration that outlives the subagent).

## 2.7 Build and Distribution

The codebase is bundled by Bun into a single JavaScript file (`cli.js`), with compile-time feature flags eliminating dead code paths. The npm package structure:

```
@anthropic-ai/claude-code/
├── cli.js          # Bundled application
├── cli.js.map      # Source map (THE FILE THAT LEAKED)
├── vendor/         # Bundled native modules
└── package.json
```

Internal ("ant") builds include additional tools and features:
- `REPLTool` — a VM-based tool that wraps other tools
- `TungstenTool` — internal-only tool
- `ConfigTool` — configuration management
- `SuggestBackgroundPRTool` — auto-PR suggestions

External builds exclude these via `process.env.USER_TYPE === 'ant'` checks and `feature()` flags.

## 2.8 Analytics and Telemetry

The codebase contains extensive analytics instrumentation under the `tengu_` prefix (Tengu being Anthropic's internal codename for Claude Code). Events are logged via:

```typescript
import { logEvent } from '../services/analytics/index.js'

logEvent('tengu_tool_use_cancelled', {
  messageID: messageId,
  toolName: sanitizeToolNameForAnalytics(tool.name),
})
```

Analytics are powered by GrowthBook for feature gating and a custom first-party event logging system. The analytics can be disabled via user settings.

## 2.9 Architectural Patterns

Several architectural patterns recur throughout the codebase:

### Lazy Require (Circular Dependency Breaking)
```typescript
const getTeamCreateTool = () =>
  require('./tools/TeamCreateTool/TeamCreateTool.js')
    .TeamCreateTool as typeof import('./tools/TeamCreateTool/TeamCreateTool.js').TeamCreateTool
```

### Feature-Gated Conditional Imports
```typescript
const coordinatorModeModule = feature('COORDINATOR_MODE')
  ? require('./coordinator/coordinatorMode.js')
  : null
```

### Memoized Async Functions
```typescript
export const getSkillDirCommands = memoize(async (cwd: string) => { ... })
```

### AsyncGenerator-Based Streaming
```typescript
async *submitMessage(prompt): AsyncGenerator<SDKMessage, void, unknown> {
  for await (const message of query({ ... })) {
    yield* normalizeMessage(message)
  }
}
```

These patterns reflect a codebase that has grown organically but maintains clear module boundaries through type-safe interfaces.

---

*Next: [Chapter 3 — Tool System](03-tool-system.md)*
