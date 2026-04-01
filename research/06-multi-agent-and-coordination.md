# Chapter 6: Multi-Agent and Coordination

## 6.1 Overview

Claude Code implements a sophisticated multi-agent system with three distinct architectural patterns:

1. **AgentTool** — The original subagent system: spawn child agents that run in isolated contexts
2. **Team/Swarm Mode** — TeamCreateTool/TeamDeleteTool for creating multi-agent teams with shared state
3. **Coordinator Mode** — A dedicated orchestration mode where Claude acts as a coordinator, delegating to workers

These represent an evolution from simple subagent spawning to full multi-agent orchestration.

## 6.2 AgentTool — The Foundation

Located in `tools/AgentTool/`, the AgentTool is the primitive for spawning subagents:

### File Structure
```
tools/AgentTool/
├── AgentTool.tsx          # Main tool implementation (React component)
├── agentColorManager.ts   # Color assignment for agent UIs
├── agentDisplay.ts        # Agent display formatting
├── agentMemorySnapshot.ts # Memory snapshots for agents
├── agentMemory.ts         # Agent memory management
├── agentToolUtils.ts      # Shared utilities
├── built-in/              # Built-in agent definitions
├── builtInAgents.ts       # Built-in agent registry
├── constants.ts           # Constants (AGENT_TOOL_NAME)
├── forkSubagent.ts        # Fork-based subagent spawning
├── loadAgentsDir.ts       # Load custom agent definitions
├── prompt.ts              # Agent system prompt
├── resumeAgent.ts         # Resume interrupted agents
├── runAgent.ts            # Core agent execution
└── UI.tsx                 # Agent UI components
```

### Agent Types

Agents have types that determine their capabilities:

```typescript
type AgentDefinition = {
  name: string
  type: string  // e.g., "worker", "researcher", "test-runner"
  description: string
  // Custom prompt addendum
  // Tool restrictions
  // Model override
}
```

Agent definitions come from:
1. **Built-in agents** — Hardcoded in `builtInAgents.ts`
2. **Custom agents** — Loaded from `.claude/agents/` directories via `loadAgentsDir.ts`
3. **Tool-specified** — Agent type passed in the tool call

### Agent Execution (`runAgent.ts`)

When AgentTool is called, `runAgent.ts` handles the execution:

```
1. Create subagent context (createSubagentContext)
   - Fork state from parent
   - Set up isolated abortController
   - Configure permission context
   - Set agentId and agentType
2. Optionally create git worktree (for isolation)
3. Set up agent memory (agentMemorySnapshot)
4. Enter the query loop (same query.ts as main thread)
5. Agent executes tools with its own permission context
6. On completion, return summary to parent
```

Key insight: **agents use the same `query()` function as the main thread**. They're recursive — agents can spawn agents.

### Fork Subagents (`forkSubagent.ts`)

The `FORK_SUBAGENT` feature flag gates a more efficient subagent mechanism:

```typescript
const forkCmd = feature('FORK_SUBAGENT')
  ? require('./commands/fork/index.js').default
  : null
```

Fork subagents share the parent's rendered system prompt bytes to maximize prompt cache hits. The parent's `renderedSystemPrompt` is frozen at fork time to prevent divergence from GrowthBook flag changes during execution.

### Agent Restrictions

Agents have tool restrictions defined in `constants/tools.ts`:

```typescript
export const ALL_AGENT_DISALLOWED_TOOLS = [...]
export const CUSTOM_AGENT_DISALLOWED_TOOLS = [...]
export const ASYNC_AGENT_ALLOWED_TOOLS = new Set([...])
```

Workers in coordinator mode get a restricted tool set — they can use Bash, Read, Edit, and MCP tools but not spawn their own agents or create teams.

## 6.3 Team/Swarm Mode

### TeamCreateTool

Creates agent swarms with shared coordination:

```typescript
export const TeamCreateTool: Tool<InputSchema, Output> = buildTool({
  name: TEAM_CREATE_TOOL_NAME,
  searchHint: 'create a multi-agent swarm team',
  maxResultSizeChars: 100_000,
  shouldDefer: true,
  // ...
})

const inputSchema = z.strictObject({
  team_name: z.string(),
  description: z.string().optional(),
  agent_type: z.string().optional(),
})
```

When a team is created:
1. Generate a unique team name (with slug collision avoidance)
2. Write a team file (`teamHelpers.ts`) recording the team structure
3. Register the team for session cleanup
4. Assign colors to teammates (`teammateLayoutManager.ts`)
5. Set up the task list for coordination
6. Initialize shared state (scratchpad directory)

### Team Architecture

```
┌─────────────────────────────┐
│         Team Lead           │
│  (Coordinator Claude)       │
│  Tools: AgentTool,          │
│         SendMessage,        │
│         TaskStop            │
└──────────┬──────────────────┘
           │ spawns
    ┌──────┴──────┐
    │             │
┌───▼───┐   ┌────▼──┐
│Worker │   │Worker │
│ Agent │   │ Agent │
│       │   │       │
│Tools: │   │Tools: │
│Bash   │   │Bash   │
│Read   │   │Read   │
│Edit   │   │Edit   │
│MCP    │   │MCP    │
│Skill  │   │Skill  │
└───────┘   └───────┘
```

### Git Worktree Isolation

Each teammate can optionally work in an isolated git worktree:

```typescript
import { createAgentWorktree, removeAgentWorktree } from '../utils/worktree.js'
```

This is gated behind `isWorktreeModeEnabled()` and provides:
- Independent file modifications without merge conflicts
- Automatic merge back to the main branch after tests pass
- Clean rollback if the worker's changes fail

### Teammate Communication

Teammates communicate via:
1. **Task system** — Shared task list with status updates
2. **SendMessageTool** — Direct messages between agents
3. **Task notifications** — XML-formatted notifications delivered as user messages

```xml
<task-notification>
<task-id>{agentId}</task-id>
<status>completed|failed|killed</status>
<summary>{human-readable status summary}</summary>
<result>{agent's final text response}</result>
<usage>
  <total_tokens>N</total_tokens>
  <tool_uses>N</tool_uses>
  <duration_ms>N</duration_ms>
</usage>
</task-notification>
```

## 6.4 Coordinator Mode

The most sophisticated multi-agent pattern, gated behind `COORDINATOR_MODE`:

```typescript
// coordinator/coordinatorMode.ts
export function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
  }
  return false
}
```

### Coordinator System Prompt

The coordinator gets a specialized system prompt via `getCoordinatorSystemPrompt()`. This is the full coordinator prompt found in the leaked source — a detailed document that defines:

1. **Role**: "You are a **coordinator**. Your job is to direct workers to research, implement, and verify code changes."
2. **Available tools**: Only AgentTool, SendMessageTool, TaskStopTool
3. **Worker capabilities**: Workers have standard tools, MCP tools, and skills
4. **Task workflow phases**:
   - **Research** (parallel) — Workers investigate the codebase
   - **Synthesis** (coordinator) — Read findings, craft implementation specs
   - **Implementation** (workers) — Make targeted changes
   - **Verification** (workers) — Test changes

### Key Coordinator Principles

From the leaked system prompt:

1. **"Always synthesize — your most important job"**: The coordinator must understand research results before delegating follow-up work. Never write "based on your findings."

2. **Parallelism is your superpower**: "Launch independent workers concurrently whenever possible — don't serialize work that can run simultaneously."

3. **Continue vs. Spawn**: Choose based on context overlap:
   - High overlap → Continue existing worker (`SendMessageTool`)
   - Low overlap → Spawn fresh worker (`AgentTool`)
   - Wrong approach → Stop and spawn fresh

4. **Self-contained prompts**: "Workers can't see your conversation. Every prompt must be self-contained."

### Coordinator Context Injection

The coordinator injects worker context into the system prompt:

```typescript
export function getCoordinatorUserContext(
  mcpClients: ReadonlyArray<{ name: string }>,
  scratchpadDir?: string,
): { [k: string]: string } {
  let content = `Workers spawned via the ${AGENT_TOOL_NAME} tool have access to these tools: ${workerTools}`
  if (mcpClients.length > 0) {
    content += `\n\nWorkers also have access to MCP tools from: ${serverNames}`
  }
  if (scratchpadDir && isScratchpadGateEnabled()) {
    content += `\n\nScratchpad directory: ${scratchpadDir}
Workers can read and write here without permission prompts.`
  }
  return { workerToolsContext: content }
}
```

### Scratchpad

The scratchpad directory (gated behind `tengu_scratch`) provides a shared filesystem space where workers can exchange data without permission prompts. This is a key innovation for multi-agent coordination.

### Session Mode Matching

When resuming a session, coordinator mode must match:

```typescript
export function matchSessionMode(
  sessionMode: 'coordinator' | 'normal' | undefined,
): string | undefined {
  const currentIsCoordinator = isCoordinatorMode()
  const sessionIsCoordinator = sessionMode === 'coordinator'
  if (currentIsCoordinator === sessionIsCoordinator) return undefined
  // Flip the env var to match
  if (sessionIsCoordinator) {
    process.env.CLAUDE_CODE_COORDINATOR_MODE = '1'
  } else {
    delete process.env.CLAUDE_CODE_COORDINATOR_MODE
  }
}
```

## 6.5 Permission Handling in Multi-Agent

Each agent level has its own permission handling:

### Coordinator Workers

Workers use `swarmWorkerHandler.ts` for permissions — they operate with restricted permissions and can't show interactive dialogs. Permission prompts are auto-denied (no UI context).

### Async Agents

Background agents have `shouldAvoidPermissionPrompts: true` set, meaning they can only use pre-approved tools or tools that match existing permission rules.

### Denial Tracking

Subagents that can't reach the root state store maintain local denial tracking:

```typescript
localDenialTracking?: DenialTrackingState
```

This ensures the fallback-to-prompting threshold is still tracked even when `setAppState` is a no-op for async agents.

## 6.6 Background Tasks

The task system (`tasks/`, `tools/Task*Tool/`) provides a structured way to track agent work:

```typescript
type Task = {
  id: string
  description: string
  status: 'pending' | 'running' | 'completed' | 'failed' | 'killed'
  agentId?: string
  result?: string
  usage?: TaskUsage
}
```

Tasks are visible in the coordinator UI and can be:
- Created (`TaskCreateTool`)
- Listed (`TaskListTool`)
- Updated (`TaskUpdateTool`)
- Stopped (`TaskStopTool`)
- Queried for output (`TaskOutputTool`)

## 6.7 Historical Evolution

The multi-agent system evolved through several phases:

1. **Early 2025**: Basic AgentTool for simple subagent spawning
2. **Jan 2026**: TeammateTool discovered in the binary (feature-flagged)
3. **Feb 2026**: Official team/swarm mode launch
4. **Mar 2026 (leak)**: Full coordinator mode with synthesized prompts visible

This progression shows Anthropic's deliberate journey from simple delegation to sophisticated orchestration — each layer building on the primitives below.

---

*Next: [Chapter 7 — IDE Bridge and Plugins](07-ide-bridge-and-plugins.md)*
