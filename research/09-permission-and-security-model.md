# Chapter 9: Permission and Security Model

## 9.1 Overview

Claude Code's permission system is a multi-layered defense-in-depth architecture. Every tool call must pass through permission checks before execution. The system balances security (preventing unauthorized actions) with usability (not interrupting the user with constant approval dialogs).

## 9.2 Permission Modes

The system supports several permission modes:

```typescript
type PermissionMode =
  | 'default'            // Normal interactive mode
  | 'plan'               // Plan mode (read-only)
  | 'bypassPermissions'  // All permissions auto-approved
  | 'auto'               // Classifier-based auto-approval
```

### Default Mode

The standard mode. Each tool call is checked against permission rules:
1. Deny rules checked first (hard block)
2. Allow rules checked next (auto-approve)
3. If neither matches, interactive prompt shown

### Plan Mode

A restricted mode where only read-only tools are allowed:

```typescript
// EnterPlanModeTool — model can enter plan mode
// ExitPlanModeV2Tool — model exits plan mode
```

When entering plan mode, the previous permission mode is saved in `prePlanMode` so it can be restored on exit.

### Bypass Permissions Mode

All permissions are auto-approved. Available based on:

```typescript
isBypassPermissionsModeAvailable: boolean
```

This is the mode used by `--dangerously-skip-permissions` and internal builds.

### Auto Mode

Classifier-based auto-approval, primarily for BashTool commands.

## 9.3 Permission Rules

Rules are organized by source and type:

```typescript
type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource
  alwaysDenyRules: ToolPermissionRulesBySource
  alwaysAskRules: ToolPermissionRulesBySource
  isBypassPermissionsModeAvailable: boolean
  isAutoModeAvailable?: boolean
  strippedDangerousRules?: ToolPermissionRulesBySource
  shouldAvoidPermissionPrompts?: boolean
  awaitAutomatedChecksBeforeDialog?: boolean
  prePlanMode?: PermissionMode
}>
```

### Rule Sources

Rules come from multiple sources, each tracked separately:

```typescript
type ToolPermissionRulesBySource = {
  [source: string]: string[]  // source → array of rule patterns
}
```

Sources include:
- `command` — Rules set by slash commands
- `user` — User-configured rules
- `project` — Project-level rules (.claude/settings)
- `policy` — Enterprise/managed settings
- `hook` — Rules set by hooks

### Rule Matching

Rules use a pattern system that matches tool names and optionally tool inputs:

```
"BashTool"           — Match all BashTool calls
"BashTool(git *)"    — Match BashTool with commands starting with "git"
"mcp__server"        — Match all tools from an MCP server
"FileWriteTool"      — Match all file writes
```

The `getDenyRuleForTool()` function checks if a tool matches any deny rules:

```typescript
export function filterToolsByDenyRules(tools, permissionContext): Tools {
  return tools.filter(tool => !getDenyRuleForTool(permissionContext, tool))
}
```

Blanket deny rules (no pattern content) actually remove tools from the model's tool set entirely — the model never sees them.

### Pattern Matching

Tools implement `preparePermissionMatcher()` for pattern matching:

```typescript
preparePermissionMatcher?(input): Promise<(pattern: string) => boolean>
```

This is called once per hook-input pair, and returns a closure for matching against patterns. For BashTool, this enables patterns like `"git *"` to match specific commands.

## 9.4 Permission Check Flow

The complete flow (from `hooks/toolPermission/PermissionContext.ts`):

```
Tool Call Received
       │
       ▼
┌──────────────────┐
│ 1. Deny Rules    │ ← Blanket deny? → Block entirely
│    (step 1a)     │ ← Pattern deny? → Reject with message
└──────┬───────────┘
       │ (not denied)
       ▼
┌──────────────────┐
│ 2. Allow Rules   │ ← Blanket allow? → Auto-approve
│    (step 1b)     │ ← Pattern allow? → Auto-approve
└──────┬───────────┘
       │ (not allowed)
       ▼
┌──────────────────┐
│ 3. PreToolUse    │ ← Hooks can approve/deny/modify input
│    Hooks         │
└──────┬───────────┘
       │ (no hook decision)
       ▼
┌──────────────────┐
│ 4. Classifier    │ ← BashTool only: ML classifier for
│    Auto-Approval │   command safety analysis
└──────┬───────────┘
       │ (no classifier decision)
       ▼
┌──────────────────┐
│ 5. Interactive   │ ← Show dialog, user approves/denies
│    Permission    │   with optional "always allow" rules
│    Dialog        │
└──────┬───────────┘
       │ (user decision)
       ▼
┌──────────────────┐
│ 6. Permission    │ ← Final hooks get to modify the
│    Request Hooks │   decision
└──────────────────┘
```

## 9.5 The Bash Classifier

The most interesting security component is the Bash command classifier, gated behind `feature('BASH_CLASSIFIER')`:

```typescript
const { tryClassifier } = feature('BASH_CLASSIFIER')
  ? {
      async tryClassifier(
        pendingClassifierCheck,
        updatedInput,
      ): Promise<PermissionDecision | null> {
        if (tool.name !== BASH_TOOL_NAME || !pendingClassifierCheck) {
          return null
        }
        const classifierDecision = await awaitClassifierAutoApproval(
          pendingClassifierCheck,
          toolUseContext.abortController.signal,
          toolUseContext.options.isNonInteractiveSession,
        )
        // ...
      },
    }
  : {}
```

The classifier (`awaitClassifierAutoApproval` from `bashPermissions.ts`) analyzes bash commands to determine if they're safe to auto-approve. Safe commands (like `git status`, `ls`, `cat`) are approved without user interaction; potentially dangerous commands still require explicit approval.

### Transcript Classifier

An additional layer, `TRANSCRIPT_CLASSIFIER`, uses the conversation transcript to make classifier decisions more context-aware:

```typescript
if (feature('TRANSCRIPT_CLASSIFIER') && classifierDecision.type === 'classifier') {
  const matchedRule = classifierDecision.reason.match(
    /^Allowed by prompt rule: "(.+)"$/,
  )?.[1]
  if (matchedRule) {
    setClassifierApproval(toolUseID, matchedRule)
  }
}
```

## 9.6 Permission Handlers

Three handlers in `hooks/toolPermission/handlers/`:

### `interactiveHandler.ts`

The default handler for REPL mode. Shows the permission dialog in the terminal UI, allowing the user to:
- Allow once
- Allow always (creates a permanent rule)
- Deny once
- Deny always
- Edit input before approving
- Provide feedback on denial

### `coordinatorHandler.ts`

For coordinator mode workers. Workers can't show interactive dialogs, so permissions are handled differently — typically through pre-configured rules or auto-denial with escalation.

### `swarmWorkerHandler.ts`

For swarm workers. Similar to coordinator handler but with swarm-specific logic for team coordination.

## 9.7 Denial Tracking

The system tracks permission denials to improve UX:

```typescript
type DenialTrackingState = {
  // Tracks denial counts per tool
  // After threshold, falls back to prompting
}
```

If a tool is denied too many times, the system may:
1. Show the user a more detailed explanation
2. Suggest creating a permanent rule
3. Escalate to a different permission mode

For subagents with no-op `setAppState`, local denial tracking prevents the counter from being lost:

```typescript
localDenialTracking?: DenialTrackingState
```

## 9.8 Permission Persistence

Permission updates can be persisted to various destinations:

```typescript
import {
  applyPermissionUpdates,
  persistPermissionUpdates,
  supportsPersistence,
} from '../../utils/permissions/PermissionUpdate.js'
```

Updates are applied both in-memory (immediate effect) and to disk (persistent across sessions). The `PermissionUpdate` type supports:
- Adding allow rules
- Adding deny rules
- Modifying mode
- Adding working directories

## 9.9 Security Boundaries

### File System Restrictions

Tools operate within defined boundaries:
- Working directory (CWD) and descendants
- Additional working directories (via `--add-dir`)
- Home directory for configuration files
- Scratchpad directory (for coordinator workers)

```typescript
type AdditionalWorkingDirectory = {
  path: string
  // Additional metadata
}
```

### Bash Security

`BashTool/bashSecurity.ts` implements:
- Command parsing for safety analysis
- Path validation (`pathValidation.ts`)
- Read-only validation (`readOnlyValidation.ts`)
- Sed edit parsing (`sedEditParser.ts`, `sedValidation.ts`)
- Destructive command warnings (`destructiveCommandWarning.ts`)
- Sandbox mode (`shouldUseSandbox.ts`)
- Mode validation (`modeValidation.ts`)

### Anti-Distillation

The fake tool injection system:

```typescript
// Gated behind tengu_anti_distill_fake_tool_injection
// Inserts fake tools into system prompt for first-party sessions
// Detects if competitors are training on Claude Code outputs
```

This is a defensive measure against model distillation — if a competitor's model starts exhibiting knowledge of fake tools that only exist in Claude Code's prompts, it indicates training data contamination.

### Content Replacement

Large tool results are replaced with file references:

```typescript
contentReplacementState?: ContentReplacementState
```

This prevents sensitive content from persisting in the conversation context longer than necessary, and ensures tool results are bounded.

## 9.10 The Trust Model

The overall trust model follows these principles:

1. **Trust on first use** — First-run trust dialog (`TrustDialog.tsx`)
2. **Least privilege** — Tools start with no permissions; rules build up
3. **User in the loop** — Interactive approval for unknown actions
4. **Progressive trust** — "Always allow" creates permanent rules
5. **Defense in depth** — Multiple layers (rules → hooks → classifier → prompt)
6. **Fail-safe** — Unknown situations default to asking the user
7. **Subagent isolation** — Subagents have restricted permissions
8. **Background denial** — Background agents can't prompt, so they fail gracefully

### The `shouldAvoidPermissionPrompts` Flag

This flag is critical for non-interactive contexts:

```typescript
shouldAvoidPermissionPrompts?: boolean
```

When true (e.g., background agents, coordinator workers), permission prompts are auto-denied. The agent must operate within pre-approved rules only.

### Coordinator Mode Awareness

In coordinator mode, permissions are checked at both levels:
1. Coordinator's permission context (for spawning workers)
2. Worker's permission context (for executing tools)

Workers inherit a restricted version of the coordinator's rules, with additional restrictions on what tools they can call.

---

*Next: [Chapter 10 — Implications and Comparison](10-implications-and-comparison.md)*
