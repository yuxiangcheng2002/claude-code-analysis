# Chapter 8: Memory, Skills, and Proactive Features

## 8.1 The Memory System (memdir/)

Claude Code's memory system is one of its most sophisticated subsystems. Located in `memdir/`, it provides a file-based, typed memory system that persists across sessions.

### Architecture

```
memdir/
├── memdir.ts             # Core memory prompt building
├── memoryScan.ts         # Memory directory scanning
├── memoryAge.ts          # Memory aging/pruning
├── memoryTypes.ts        # Memory type taxonomy
├── paths.ts              # Memory directory paths
├── findRelevantMemories.ts # Memory retrieval
├── teamMemPaths.ts       # Team memory paths (TEAMMEM)
└── teamMemPrompts.ts     # Team memory prompts (TEAMMEM)
```

### Memory Types

The system enforces a **closed four-type taxonomy**:

1. **User** — Facts about the user (role, preferences, communication style)
2. **Feedback** — Corrections and behavioral guidance ("use bun, not npm")
3. **Project** — Non-derivable project context (deadlines, decisions, rationale)
4. **Reference** — Pointers to external systems (dashboards, URLs, channels)

Explicitly excluded: information derivable from the current project state (code patterns, architecture, git history).

### Memory Storage

Memory is file-based with a specific structure:

```
~/.claude/projects/<project-slug>/memory/
├── MEMORY.md           # Index file (max 200 lines, 25KB)
├── user_role.md        # Individual memory files
├── feedback_testing.md
├── project_deadline.md
└── logs/               # Daily logs (KAIROS mode)
    └── 2026/
        └── 03/
            └── 2026-03-31.md
```

### MEMORY.md — The Index

`MEMORY.md` is always loaded into context. It serves as an index, not a memory store:

```typescript
export const ENTRYPOINT_NAME = 'MEMORY.md'
export const MAX_ENTRYPOINT_LINES = 200
export const MAX_ENTRYPOINT_BYTES = 25_000
```

Each entry should be one line, under ~150 characters:
```
- [User Role](user_role.md) — Senior backend eng, prefers Rust
```

The `truncateEntrypointContent()` function enforces both line and byte caps, with warnings when truncation occurs:

```typescript
export function truncateEntrypointContent(raw: string): EntrypointTruncation {
  const wasLineTruncated = lineCount > MAX_ENTRYPOINT_LINES
  const wasByteTruncated = byteCount > MAX_ENTRYPOINT_BYTES
  // Truncation warning appended if either cap is hit
}
```

### Memory Prompt

The memory prompt injected into the system prompt is built by `buildMemoryLines()`:

```typescript
export function buildMemoryLines(
  displayName: string,
  memoryDir: string,
  extraGuidelines?: string[],
  skipIndex = false,
): string[]
```

The prompt includes:
1. Description of the memory system and its location
2. Type taxonomy with examples
3. What NOT to save (derivable info)
4. How to save memories (two-step: write file + update index)
5. When to access memories
6. "Trusting recall" section
7. Searching past context section

### Memory Frontmatter

Each memory file uses YAML frontmatter:

```yaml
---
name: User Role
description: User is a senior backend engineer
type: user
---
```

### Team Memory (TEAMMEM)

Gated behind `feature('TEAMMEM')`, team memory enables shared memory across team members:

```typescript
const teamMemPaths = feature('TEAMMEM')
  ? require('./teamMemPaths.js')
  : null
```

Team memory lives under `memory/team/` and is synchronized via `teamMemorySync` service.

### Searching Past Context

When the `tengu_coral_fern` feature gate is enabled, the memory prompt includes instructions for searching past context:

```typescript
export function buildSearchingPastContextSection(autoMemDir: string): string[] {
  // 1. Search topic files in memory directory (grep)
  // 2. Session transcript logs (last resort — large files)
}
```

## 8.2 The Skill System

Skills are Claude Code's extensible command framework, loaded from `skills/`:

### Skill Loading Pipeline

```
skills/
├── loadSkillsDir.ts       # Main skill loader (~700 lines)
├── bundledSkills.ts       # Built-in skill registry
├── bundled/               # Built-in skill implementations
└── mcpSkillBuilders.ts    # MCP-to-skill bridge
```

The loader searches multiple directories:

```typescript
export const getSkillDirCommands = memoize(async (cwd: string) => {
  // 1. Managed (policy) skills: ~/.claude-managed/.claude/skills/
  // 2. User skills: ~/.claude/skills/
  // 3. Project skills: .claude/skills/ (walks up to home)
  // 4. Additional directories: --add-dir paths
  // 5. Legacy commands: .claude/commands/ (deprecated)
})
```

### Skill Format

Skills use the `SKILL.md` format with YAML frontmatter:

```markdown
---
name: My Skill
description: Does something useful
when_to_use: When the user asks to do X
allowed-tools: [BashTool, FileReadTool]
model: claude-sonnet-4-20250514
effort: high
context: fork
agent: researcher
user-invocable: true
paths: ["src/auth/**"]
---

# My Skill

Instructions for the model...

${CLAUDE_SKILL_DIR} — replaced with skill directory path
${CLAUDE_SESSION_ID} — replaced with session ID
```

### Conditional Skills

Skills can be activated only when relevant files are touched:

```typescript
export function activateConditionalSkillsForPaths(
  filePaths: string[],
  cwd: string,
): string[]
```

A skill with `paths: ["src/auth/**"]` will only be loaded into the tool set when the user reads or writes files matching that pattern. This keeps the prompt lean until the skill becomes relevant.

### Dynamic Skill Discovery

Skills are discovered on-the-fly during file operations:

```typescript
export async function discoverSkillDirsForPaths(
  filePaths: string[],
  cwd: string,
): Promise<string[]>
```

When a tool operates on a file, the system walks up from the file's directory to CWD looking for `.claude/skills/` directories. Newly discovered skills are loaded and made available immediately — without restarting the session.

### Shell Commands in Skills

Skills can embed shell commands that execute during prompt generation:

```markdown
Current git status: !`git status --short`
```

These `!` backtick blocks are executed by `executeShellCommandsInPrompt()` — but only for local skills. **MCP skills are explicitly blocked from inline shell execution** for security:

```typescript
if (loadedFrom !== 'mcp') {
  finalContent = await executeShellCommandsInPrompt(finalContent, ...)
}
```

### Skill Deduplication

Skills from multiple sources are deduplicated by resolved file path (via `realpath`):

```typescript
async function getFileIdentity(filePath: string): Promise<string | null> {
  try {
    return await realpath(filePath)
  } catch {
    return null
  }
}
```

This handles symlinks and overlapping parent directories — a skill accessed through two different paths is only loaded once.

## 8.3 KAIROS — The Proactive Assistant

KAIROS is Claude Code's most ambitious unreleased feature: a persistent, autonomous assistant mode. It's gated behind `feature('KAIROS')`.

### What KAIROS Is

From the leaked source (and external analysis by WaveSpeed AI and Engineer's Codex):

- **Persistent sessions** — KAIROS sessions don't end when the terminal closes
- **Daily logging** — Append-only daily log files instead of MEMORY.md index
- **Proactive behavior** — The agent watches, logs, and acts without being asked
- **Dream mode** — Nightly memory consolidation process
- **Push notifications** — Send notifications to the user via `PushNotificationTool`
- **File sending** — Send files to the user via `SendUserFileTool`
- **Sleep scheduling** — `SleepTool` for timed delays between proactive actions

### Daily Log Architecture

In KAIROS mode, memory uses a different paradigm:

```typescript
function buildAssistantDailyLogPrompt(skipIndex = false): string {
  const logPathPattern = join(memoryDir, 'logs', 'YYYY', 'MM', 'YYYY-MM-DD.md')
  // Append-only daily log files
  // Nightly /dream skill distills logs into MEMORY.md
}
```

Instead of maintaining MEMORY.md as a live index, the agent writes timestamped entries to daily log files. A separate nightly "dreaming" process (accessible via `/dream`) consolidates these logs into topic files and updates MEMORY.md.

### PROACTIVE Feature Flag

The `PROACTIVE` flag enables a subset of KAIROS capabilities:

```typescript
const SleepTool =
  feature('PROACTIVE') || feature('KAIROS')
    ? require('./tools/SleepTool/SleepTool.js').SleepTool
    : null
```

PROACTIVE enables SleepTool and basic background agent capabilities without the full KAIROS assistant mode.

### BUDDY

Another unreleased companion feature:

```typescript
const buddy = feature('BUDDY')
  ? require('./commands/buddy/index.js').default
  : null
```

Details are sparse in the leaked source — the `buddy/` directory under `src/` suggests a companion agent system, but the implementation is not fully visible.

## 8.4 Cron and Scheduling

### Agent Triggers

The `AGENT_TRIGGERS` feature flag enables cron-based scheduling:

```typescript
const cronTools = feature('AGENT_TRIGGERS')
  ? [CronCreateTool, CronDeleteTool, CronListTool]
  : []
```

Three tools for managing scheduled tasks:
- `CronCreateTool` — Schedule recurring tasks
- `CronDeleteTool` — Remove scheduled tasks
- `CronListTool` — List active schedules

### Remote Triggers

`AGENT_TRIGGERS_REMOTE` extends this to remote triggers:

```typescript
const RemoteTriggerTool = feature('AGENT_TRIGGERS_REMOTE')
  ? require('./tools/RemoteTriggerTool/RemoteTriggerTool.js').RemoteTriggerTool
  : null
```

### Scheduled Tasks Hook

The REPL includes a `useScheduledTasks.ts` hook that manages the lifecycle of scheduled tasks, ensuring they run at the right intervals and clean up properly.

## 8.5 Voice Mode

Gated behind `feature('VOICE_MODE')`:

```typescript
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null
```

Supporting files:
- `voice/voiceModeEnabled.ts` — Feature detection
- `services/voiceStreamSTT.ts` — Speech-to-text streaming
- `services/voiceKeyterms.ts` — Key term detection
- `services/voice.ts` — Voice service orchestration
- `hooks/useVoice.ts` — Voice integration hook
- `hooks/useVoiceIntegration.tsx` — Voice UI integration
- `hooks/useVoiceEnabled.ts` — Feature check hook
- `components/LogoV2/VoiceModeNotice.tsx` — Voice mode UI notice

## 8.6 Workflow Scripts

Gated behind `feature('WORKFLOW_SCRIPTS')`:

```typescript
const WorkflowTool = feature('WORKFLOW_SCRIPTS')
  ? (() => {
      require('./tools/WorkflowTool/bundled/index.js').initBundledWorkflows()
      return require('./tools/WorkflowTool/WorkflowTool.js').WorkflowTool
    })()
  : null
```

Workflows provide multi-step automation with bundled workflow definitions. The `initBundledWorkflows()` call at import time registers built-in workflows.

## 8.7 Context Management Features

### History Snip

`HISTORY_SNIP` enables aggressive context window management:

```typescript
const SnipTool = feature('HISTORY_SNIP')
  ? require('./tools/SnipTool/SnipTool.js').SnipTool
  : null
```

Snipping allows marking points in the conversation where older history can be safely removed, keeping only a summary. This is more aggressive than auto-compaction.

### Context Collapse

`CONTEXT_COLLAPSE` provides an even more aggressive strategy:

```typescript
const CtxInspectTool = feature('CONTEXT_COLLAPSE')
  ? require('./tools/CtxInspectTool/CtxInspectTool.js').CtxInspectTool
  : null
```

The `CtxInspectTool` allows inspecting the current context state, presumably to inform decisions about what to collapse.

### Reactive Compaction

`REACTIVE_COMPACT` triggers compaction based on context pressure:

```typescript
const reactiveCompact = feature('REACTIVE_COMPACT')
  ? require('./services/compact/reactiveCompact.js')
  : null
```

Rather than waiting for a fixed token threshold, reactive compaction responds to context window pressure dynamically.

---

*Next: [Chapter 9 — Permission and Security Model](09-permission-and-security-model.md)*
