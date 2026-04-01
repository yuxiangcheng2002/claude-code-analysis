# Chapter 3: The Tool System

## 3.1 Overview

The tool system is the beating heart of Claude Code. Every capability — reading files, running commands, searching the web, spawning sub-agents — is implemented as a discrete, permission-gated **Tool**. The system is defined across two primary files:

- **`Tool.ts`** (792 lines) — Type definitions, interfaces, the `Tool<Input, Output, Progress>` generic type
- **`tools.ts`** (389 lines) — Tool registry, assembly, filtering, and presets

## 3.2 The Tool Interface

Every tool implements the `Tool<Input, Output, Progress>` generic interface. Here are the most important methods:

```typescript
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  readonly name: string
  aliases?: string[]
  searchHint?: string  // For ToolSearch keyword matching

  // Core execution
  call(args, context, canUseTool, parentMessage, onProgress?): Promise<ToolResult<Output>>

  // Schema and validation
  readonly inputSchema: Input  // Zod schema
  readonly inputJSONSchema?: ToolInputJSONSchema  // For MCP tools
  outputSchema?: z.ZodType<unknown>
  validateInput?(input, context): Promise<ValidationResult>

  // Permission system
  checkPermissions(input, context): Promise<PermissionResult>
  preparePermissionMatcher?(input): Promise<(pattern: string) => boolean>

  // Behavioral metadata
  isConcurrencySafe(input): boolean
  isReadOnly(input): boolean
  isDestructive?(input): boolean
  isEnabled(): boolean
  interruptBehavior?(): 'cancel' | 'block'

  // System prompt
  description(input, options): Promise<string>
  prompt(options): Promise<string>

  // UI rendering (React/Ink)
  renderToolUseMessage(input, options): React.ReactNode
  renderToolResultMessage?(content, progress, options): React.ReactNode
  renderToolUseProgressMessage?(progress, options): React.ReactNode
  renderToolUseRejectedMessage?(input, options): React.ReactNode
  renderGroupedToolUse?(toolUses, options): React.ReactNode | null

  // Metadata
  maxResultSizeChars: number
  readonly shouldDefer?: boolean   // Deferred loading via ToolSearch
  readonly alwaysLoad?: boolean    // Never defer
  readonly strict?: boolean        // Strict schema adherence
  mcpInfo?: { serverName: string; toolName: string }  // MCP tools

  // Mapping and serialization
  mapToolResultToToolResultBlockParam(content, toolUseID): ToolResultBlockParam
  toAutoClassifierInput(input): unknown
  backfillObservableInput?(input): void
}
```

### Key Design Decisions

1. **Every tool is a React component**: Tools have `render*` methods that return `React.ReactNode`. The entire UI is React/Ink-based, so even tool output is rendered as React components.

2. **Permission is first-class**: `checkPermissions()` is a required method. Every tool must declare its permission requirements.

3. **Concurrency awareness**: `isConcurrencySafe()` lets the system run safe tools in parallel while serializing unsafe ones.

4. **Deferred loading**: Tools marked `shouldDefer: true` aren't sent to the model initially — they require `ToolSearch` first, reducing prompt size.

5. **Read-only distinction**: `isReadOnly()` enables parallel execution and reduced permission requirements for read operations.

## 3.3 Complete Tool Inventory

Here is every tool found in the source, organized by category:

### File Operations

| Tool | File | Description |
|------|------|-------------|
| `FileReadTool` | `tools/FileReadTool/` | Read file contents with line limits |
| `FileWriteTool` | `tools/FileWriteTool/` | Write content to files |
| `FileEditTool` | `tools/FileEditTool/` | Surgical text edits (find/replace) |
| `NotebookEditTool` | `tools/NotebookEditTool/` | Jupyter notebook editing |
| `GlobTool` | `tools/GlobTool/` | File pattern matching |
| `GrepTool` | `tools/GrepTool/` | Content search (ripgrep-based) |

### Execution

| Tool | File | Description |
|------|------|-------------|
| `BashTool` | `tools/BashTool/` | Shell command execution |
| `PowerShellTool` | `tools/PowerShellTool/` | PowerShell on Windows (conditional) |
| `REPLTool` | `tools/REPLTool/` | VM-based REPL wrapper (ant-only) |

### Web & External

| Tool | File | Description |
|------|------|-------------|
| `WebFetchTool` | `tools/WebFetchTool/` | Fetch web pages |
| `WebSearchTool` | `tools/WebSearchTool/` | Web search |
| `WebBrowserTool` | `tools/WebBrowserTool/` | Browser automation (feature-gated) |

### Agent & Multi-Agent

| Tool | File | Description |
|------|------|-------------|
| `AgentTool` | `tools/AgentTool/` | Spawn sub-agents |
| `TeamCreateTool` | `tools/TeamCreateTool/` | Create agent swarms |
| `TeamDeleteTool` | `tools/TeamDeleteTool/` | Delete agent teams |
| `SendMessageTool` | `tools/SendMessageTool/` | Send messages to agents |

### Task Management

| Tool | File | Description |
|------|------|-------------|
| `TaskCreateTool` | `tools/TaskCreateTool/` | Create background tasks |
| `TaskGetTool` | `tools/TaskGetTool/` | Get task status |
| `TaskUpdateTool` | `tools/TaskUpdateTool/` | Update task state |
| `TaskListTool` | `tools/TaskListTool/` | List all tasks |
| `TaskStopTool` | `tools/TaskStopTool/` | Stop running tasks |
| `TaskOutputTool` | `tools/TaskOutputTool/` | Get task output |
| `TodoWriteTool` | `tools/TodoWriteTool/` | Write to todo list |

### Planning & Mode

| Tool | File | Description |
|------|------|-------------|
| `EnterPlanModeTool` | `tools/EnterPlanModeTool/` | Enter plan mode |
| `ExitPlanModeV2Tool` | `tools/ExitPlanModeTool/` | Exit plan mode |
| `EnterWorktreeTool` | `tools/EnterWorktreeTool/` | Enter git worktree isolation |
| `ExitWorktreeTool` | `tools/ExitWorktreeTool/` | Exit git worktree |

### MCP & External Tools

| Tool | File | Description |
|------|------|-------------|
| `MCPTool` | `tools/MCPTool/` | MCP server tool proxy |
| `ListMcpResourcesTool` | `tools/ListMcpResourcesTool/` | List MCP resources |
| `ReadMcpResourceTool` | `tools/ReadMcpResourceTool/` | Read MCP resources |
| `McpAuthTool` | `tools/McpAuthTool/` | MCP authentication |
| `LSPTool` | `tools/LSPTool/` | Language Server Protocol integration |

### Skills & Configuration

| Tool | File | Description |
|------|------|-------------|
| `SkillTool` | `tools/SkillTool/` | Invoke skills (custom commands) |
| `ConfigTool` | `tools/ConfigTool/` | Configuration management (ant-only) |
| `ToolSearchTool` | `tools/ToolSearchTool/` | Search for available tools |
| `BriefTool` | `tools/BriefTool/` | Generate brief summaries |

### Scheduling & Automation

| Tool | File | Description |
|------|------|-------------|
| `ScheduleCronTool` | `tools/ScheduleCronTool/` | Cron job scheduling (3 sub-tools) |
| `RemoteTriggerTool` | `tools/RemoteTriggerTool/` | Remote trigger execution |
| `SleepTool` | `tools/SleepTool/` | Sleep/wait (PROACTIVE/KAIROS) |
| `MonitorTool` | `tools/MonitorTool/` | System monitoring |

### Communication

| Tool | File | Description |
|------|------|-------------|
| `AskUserQuestionTool` | `tools/AskUserQuestionTool/` | Ask user a question |
| `PushNotificationTool` | `tools/PushNotificationTool/` | Push notifications (KAIROS) |
| `SendUserFileTool` | `tools/SendUserFileTool/` | Send files to user (KAIROS) |
| `SubscribePRTool` | `tools/SubscribePRTool/` | Subscribe to PR events |

### Internal / Special

| Tool | File | Description |
|------|------|-------------|
| `TungstenTool` | `tools/TungstenTool/` | Internal Anthropic tool (ant-only) |
| `SyntheticOutputTool` | `tools/SyntheticOutputTool/` | Structured output enforcement |
| `TestingPermissionTool` | `tools/testing/` | Testing-only permission tool |
| `VerifyPlanExecutionTool` | `tools/VerifyPlanExecutionTool/` | Plan verification (env-gated) |
| `SuggestBackgroundPRTool` | `tools/SuggestBackgroundPRTool/` | Auto-PR suggestions (ant-only) |
| `OverflowTestTool` | `tools/OverflowTestTool/` | Context overflow testing |
| `CtxInspectTool` | `tools/CtxInspectTool/` | Context inspection |
| `TerminalCaptureTool` | `tools/TerminalCaptureTool/` | Terminal state capture |
| `SnipTool` | `tools/SnipTool/` | History snipping |
| `ListPeersTool` | `tools/ListPeersTool/` | List UDS peers |
| `WorkflowTool` | `tools/WorkflowTool/` | Workflow automation |

## 3.4 Tool Registry and Assembly

The `tools.ts` file manages how tools are assembled into the active tool set.

### `getAllBaseTools()` — The Source of Truth

This function returns every tool that *could* be available in the current environment:

```typescript
export function getAllBaseTools(): Tools {
  return [
    AgentTool,
    TaskOutputTool,
    BashTool,
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
    ExitPlanModeV2Tool,
    FileReadTool,
    FileEditTool,
    FileWriteTool,
    NotebookEditTool,
    WebFetchTool,
    TodoWriteTool,
    WebSearchTool,
    TaskStopTool,
    AskUserQuestionTool,
    SkillTool,
    EnterPlanModeTool,
    ...(process.env.USER_TYPE === 'ant' ? [ConfigTool] : []),
    ...(isAgentSwarmsEnabled() ? [getTeamCreateTool(), getTeamDeleteTool()] : []),
    // ... many more conditional tools
  ]
}
```

Notable: when Ant-native builds have embedded search tools (bfs/ugrep in the Bun binary), the dedicated Glob/Grep tools are excluded since shell aliases handle the same functionality.

### `getTools()` — Filtered for Use

Applies permission filtering and mode-specific logic:

```typescript
export const getTools = (permissionContext: ToolPermissionContext): Tools => {
  // Simple mode: only Bash, Read, and Edit
  if (isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)) {
    return [BashTool, FileReadTool, FileEditTool]
  }
  // ... filter by deny rules, REPL mode, isEnabled()
}
```

### `assembleToolPool()` — Full Pool with MCP

Combines built-in tools with MCP tools, maintaining sort order for prompt cache stability:

```typescript
export function assembleToolPool(permissionContext, mcpTools): Tools {
  const builtInTools = getTools(permissionContext)
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)
  // Sort each partition for prompt-cache stability
  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
    'name',
  )
}
```

The comment is revealing: built-in tools are kept as a contiguous prefix to avoid invalidating downstream cache keys when MCP tools change.

## 3.5 Tool Execution Flow

When Claude decides to call a tool, this sequence occurs:

```
1. Model returns tool_use block with name + input
2. Input parsed against tool.inputSchema (Zod)
3. tool.validateInput() called — structural validation
4. Permission check:
   a. Check deny rules (blanket + pattern)
   b. Check allow rules (blanket + pattern)
   c. Run PreToolUse hooks
   d. Classifier auto-approval (Bash only)
   e. Interactive permission dialog (if needed)
   f. Run PermissionRequest hooks
5. tool.call() executes with approved (possibly modified) input
6. PostToolUse hooks run on result
7. Result mapped to ToolResultBlockParam for API
8. Tool result budget applied (large results persisted to disk)
9. Result sent back to model as next message
```

### Concurrency and Streaming

The `StreamingToolExecutor` (in `services/tools/`) enables parallel tool execution for concurrency-safe tools. Tools declare their safety via `isConcurrencySafe()`:

- **Safe**: FileReadTool, GrepTool, GlobTool, WebFetchTool
- **Unsafe**: BashTool, FileWriteTool, FileEditTool (must serialize)

When multiple tool calls arrive in a single assistant message, safe tools run in parallel while unsafe ones execute sequentially.

## 3.6 Tool Result Budget

Large tool results are automatically persisted to disk to avoid context window bloat:

```typescript
maxResultSizeChars: number  // Per-tool limit
```

When a tool result exceeds this limit, it's saved to a file and Claude receives a truncated preview with the file path. FileReadTool sets this to `Infinity` because persisting creates a circular read loop.

## 3.7 ToolSearch — Deferred Tool Loading

With 40+ tools, sending all schemas in every prompt wastes tokens. `ToolSearchTool` enables deferred loading:

1. Tools marked `shouldDefer: true` are sent to the API with `defer_loading: true`
2. The model only sees their names, not full schemas
3. To use a deferred tool, the model first calls `ToolSearch` with keywords
4. The matched tool's full schema is then loaded

Tools can opt out with `alwaysLoad: true` — for tools the model must see on turn 1.

The `searchHint` field provides keyword matching:
```typescript
searchHint: 'create a multi-agent swarm team'  // TeamCreateTool
searchHint: 'jupyter'  // NotebookEditTool
```

## 3.8 The ToolUseContext

Every tool call receives a `ToolUseContext` — a rich context object containing:

```typescript
type ToolUseContext = {
  options: {
    commands: Command[]
    tools: Tools
    mcpClients: MCPServerConnection[]
    mainLoopModel: string
    thinkingConfig: ThinkingConfig
    agentDefinitions: AgentDefinitionsResult
    maxBudgetUsd?: number
    // ...
  }
  abortController: AbortController
  readFileState: FileStateCache
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void
  setAppStateForTasks?: // Always-shared state for infrastructure
  handleElicitation?: // MCP URL elicitation handler
  setToolJSX?: SetToolJSXFn // For tools that render custom UI
  messages: Message[]  // Full conversation history
  agentId?: AgentId   // Set for subagents
  contentReplacementState?: ContentReplacementState
  // ... many more fields
}
```

This context gives tools access to the full application state, enabling complex inter-tool coordination.

---

*Next: [Chapter 4 — Query Engine](04-query-engine.md)*
