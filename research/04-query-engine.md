# Chapter 4: The Query Engine

## 4.1 Overview

The Query Engine is the central nervous system of Claude Code. Split across two files — `QueryEngine.ts` (1,295 lines) and `query.ts` (1,729 lines) — it manages the complete lifecycle of conversations with the LLM, from system prompt assembly to streaming tool execution loops.

The division of responsibility:
- **`QueryEngine.ts`** — Session lifecycle, message management, SDK/print-mode interface
- **`query.ts`** — The actual LLM API call loop, streaming, tool execution, compaction

## 4.2 QueryEngine Class

The `QueryEngine` class is the entry point for programmatic use (SDK, print mode):

```typescript
export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]
  private abortController: AbortController
  private permissionDenials: SDKPermissionDenial[]
  private totalUsage: NonNullableUsage
  private readFileState: FileStateCache
  private discoveredSkillNames = new Set<string>()
  private loadedNestedMemoryPaths = new Set<string>()
}
```

### Configuration

```typescript
export type QueryEngineConfig = {
  cwd: string
  tools: Tools
  commands: Command[]
  mcpClients: MCPServerConnection[]
  agents: AgentDefinition[]
  canUseTool: CanUseToolFn
  getAppState: () => AppState
  setAppState: (f: (prev: AppState) => AppState) => void
  initialMessages?: Message[]
  readFileCache: FileStateCache
  customSystemPrompt?: string
  appendSystemPrompt?: string
  userSpecifiedModel?: string
  fallbackModel?: string
  thinkingConfig?: ThinkingConfig
  maxTurns?: number
  maxBudgetUsd?: number
  taskBudget?: { total: number }
  jsonSchema?: Record<string, unknown>
  snipReplay?: (msg, store) => { messages: Message[]; executed: boolean } | undefined
  // ... more options
}
```

### The submitMessage Flow

`submitMessage()` is an async generator that yields `SDKMessage` events:

```typescript
async *submitMessage(
  prompt: string | ContentBlockParam[],
  options?: { uuid?: string; isMeta?: boolean },
): AsyncGenerator<SDKMessage, void, unknown>
```

The flow:

```
1. Clear turn-scoped state (discoveredSkillNames)
2. Set CWD
3. Build system prompt:
   a. fetchSystemPromptParts() — assembles default prompt, user context, system context
   b. Inject memory mechanics prompt if custom prompt + CLAUDE_COWORK_MEMORY_PATH_OVERRIDE
   c. Concatenate: [customPrompt | defaultPrompt] + [memoryMechanics] + [appendPrompt]
4. Process user input:
   a. Run slash command detection and execution
   b. Expand attachments, images
   c. Determine allowed tools from command output
5. Push messages to mutableMessages
6. Record transcript (for resume capability)
7. Load skills and plugins (cache-only for SDK mode)
8. Yield SystemInitMessage (tools, MCP, model info)
9. If shouldQuery=false (local command), yield results and return
10. Enter query loop:
    for await (const message of query({ messages, systemPrompt, ... })) {
      // Process each message type (assistant, user, progress, stream_event, etc.)
      // Track usage, update transcript, handle compaction
    }
11. Yield final result message with usage stats
```

### System Prompt Assembly

The system prompt is assembled from multiple sources:

```typescript
const { defaultSystemPrompt, userContext, systemContext } =
  await fetchSystemPromptParts({
    tools,
    mainLoopModel,
    additionalWorkingDirectories,
    mcpClients,
    customSystemPrompt,
  })
```

User context includes:
- Current date/time
- Git status (branch, modified files)
- CLAUDE.md file contents
- Memory prompt (if auto-memory enabled)
- Coordinator mode context (worker tool list, scratchpad info)

The system prompt is wrapped in `asSystemPrompt()` which marks it as a typed system prompt to prevent accidental mutation.

## 4.3 The query() Function

This is the inner loop — the 1,729-line function that actually talks to the LLM API:

```typescript
export async function* query(params: {
  messages: Message[]
  systemPrompt: SystemPrompt
  userContext: Record<string, string>
  systemContext: Record<string, string>
  canUseTool: CanUseToolFn
  toolUseContext: ToolUseContext
  fallbackModel?: string
  querySource: QuerySource
  maxTurns?: number
  taskBudget?: { total: number }
}): AsyncGenerator<Message>
```

### The Agentic Loop

The core of query.ts is an agentic loop that continues until the model stops requesting tool calls:

```
while (true) {
  1. Normalize messages for API (strip UI-only messages, apply snip boundaries)
  2. Prepend user context, append system context
  3. Check token count → auto-compact if needed
  4. Call Anthropic API with streaming
  5. Process stream events:
     - text blocks → yield assistant messages
     - thinking blocks → yield thinking messages
     - tool_use blocks → collect for execution
  6. If no tool calls → break (conversation turn complete)
  7. Execute tools:
     - Parallel for concurrency-safe tools
     - Sequential for unsafe tools
     - Apply permission checks
     - Handle rejections/errors
  8. Push tool results as user messages
  9. Run PostSampling hooks
  10. Increment turn counter
  11. Check maxTurns / budget limits
  12. Continue loop
}
```

### Streaming

Claude Code uses the Anthropic SDK's streaming API. Stream events are processed in real-time:

```typescript
for await (const event of stream) {
  switch (event.type) {
    case 'message_start': // New message, reset usage tracking
    case 'content_block_start': // New content block (text, thinking, tool_use)
    case 'content_block_delta': // Incremental content update
    case 'content_block_stop': // Block complete
    case 'message_delta': // Final message metadata (usage, stop_reason)
    case 'message_stop': // Message complete
  }
}
```

### Auto-Compaction

When the context window gets too large, Claude Code automatically compacts the conversation:

```typescript
if (isAutoCompactEnabled()) {
  const warningState = calculateTokenWarningState(tokenCount, model)
  if (warningState.shouldAutoCompact) {
    // Trigger compaction
    const compactedMessages = await buildPostCompactMessages(messages, ...)
    messages = compactedMessages
  }
}
```

Compaction is handled by `services/compact/` which:
1. Summarizes old messages into a compact representation
2. Preserves recent messages and tool results
3. Inserts a `compact_boundary` message marking the transition
4. Optionally runs pre/post-compact hooks

There's also a "reactive compact" feature (`REACTIVE_COMPACT`) and "context collapse" (`CONTEXT_COLLAPSE`) for more aggressive context management.

### Thinking Mode

Claude Code supports extended thinking (chain-of-thought):

```typescript
type ThinkingConfig =
  | { type: 'disabled' }
  | { type: 'adaptive' }  // Model decides when to think
  | { type: 'enabled'; budgetTokens?: number }
```

When thinking is enabled, `thinking` content blocks appear in the stream alongside `text` blocks. The thinking content is displayed to the user but not sent back to the API on subsequent turns.

### Retry and Error Handling

The query loop handles several error scenarios:

1. **Rate limiting** — Exponential backoff with rate limit messages
2. **Prompt too long** — Auto-compact and retry
3. **API errors** — Categorized as retryable vs. fatal via `categorizeRetryableAPIError()`
4. **Fallback model** — If the primary model fails, falls back to `fallbackModel`
5. **Image errors** — `ImageSizeError` and `ImageResizeError` handling

```typescript
try {
  // API call
} catch (error) {
  if (error instanceof FallbackTriggeredError) {
    // Switch to fallback model
  }
  const category = categorizeRetryableAPIError(error)
  if (category === 'retryable') {
    // Retry with backoff
  }
}
```

### Snip Boundaries (HISTORY_SNIP)

When enabled, the snip system allows aggressive history truncation for long-running sessions:

```typescript
const snipModule = feature('HISTORY_SNIP')
  ? require('./services/compact/snipCompact.js')
  : null
```

Snip boundaries mark points where history can be safely truncated. The `snipReplay` callback in QueryEngine allows the REPL and SDK modes to handle snip differently — REPL keeps full history for UI scrollback, SDK truncates to bound memory.

## 4.4 Tool Execution Pipeline

Within the query loop, tool calls go through `runTools()` from `services/tools/toolOrchestration.ts`:

```
Tool Calls from Model
        │
        ▼
┌───────────────────┐
│ Parse & Validate  │ ← Zod schema validation
└───────┬───────────┘
        │
        ▼
┌───────────────────┐
│ Permission Check  │ ← Rules, hooks, classifier, interactive
└───────┬───────────┘
        │
        ▼
┌───────────────────┐
│ Parallel/Serial   │ ← Based on isConcurrencySafe()
│ Execution         │
└───────┬───────────┘
        │
        ▼
┌───────────────────┐
│ Result Processing │ ← Budget, persistence, mapping
└───────┬───────────┘
        │
        ▼
  Tool Result Messages
```

The `StreamingToolExecutor` manages concurrent execution:
- Groups tool calls by concurrency safety
- Runs safe tools in parallel via `Promise.all`
- Serializes unsafe tools
- Reports progress events during execution

## 4.5 Message Types

The message system supports a rich taxonomy:

```typescript
type Message =
  | UserMessage          // User input
  | AssistantMessage     // Model response
  | SystemMessage        // System notifications
  | AttachmentMessage    // File/memory attachments
  | ProgressMessage      // Tool progress events
  | StreamEvent          // Raw streaming events
  | TombstoneMessage     // Control signals for message removal
  | ToolUseSummaryMessage // Summarized tool usage
```

Messages are carefully typed to distinguish between API-bound messages (sent to Anthropic) and UI-only messages (displayed but stripped at the API boundary). The `normalizeMessagesForAPI()` function handles this filtering.

## 4.6 Token Tracking and Cost

Token usage is tracked at multiple granularities:

```typescript
import { accumulateUsage, updateUsage } from 'src/services/api/claude.js'
import { getTotalCost, getModelUsage, getTotalAPIDuration } from './cost-tracker.js'
```

The cost tracker maintains:
- Per-message usage (input/output/cache tokens)
- Cumulative session usage
- Per-model usage breakdown
- API duration tracking

Budget limits can be set via `maxBudgetUsd` and `taskBudget`, with automatic stopping when exceeded.

## 4.7 Session Persistence

Every message exchange is persisted to disk via `recordTranscript()`:

```typescript
if (persistSession) {
  await recordTranscript(messages)
}
```

This enables:
- **Resume** (`--resume`) — Continue a previous session
- **Export** — Export conversation history
- **Debugging** — Inspect full conversation flow

The persistence system uses a write queue with lazy JSON serialization and a 100ms drain timer to avoid blocking the main loop.

## 4.8 Coordinator Mode Integration

In coordinator mode (`COORDINATOR_MODE`), the query loop integrates with the multi-agent system:

```typescript
const getCoordinatorUserContext = feature('COORDINATOR_MODE')
  ? require('./coordinator/coordinatorMode.js').getCoordinatorUserContext
  : () => ({})
```

This injects additional context about available worker tools, MCP servers, and the scratchpad directory into the system prompt, enabling the coordinator to delegate work effectively.

---

*Next: [Chapter 5 — Command System](05-command-system.md)*
