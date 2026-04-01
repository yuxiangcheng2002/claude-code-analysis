# Chapter 5: The Command System

## 5.1 Overview

Claude Code has a dual-layer command system: **slash commands** (user-invoked via `/name`) and **skills** (model-invoked via `SkillTool`). The command registry in `commands.ts` (754 lines) imports and organizes over **100 commands** from the `commands/` directory.

## 5.2 Command Type System

Commands are typed with a discriminated union:

```typescript
type Command =
  | PromptCommand   // Generates a prompt for the model
  | LocalCommand    // Executes locally without model interaction
  | InjectedCommand // External command injected by plugins/skills
```

### PromptCommand

The most common type. Generates content that's sent as a prompt to the model:

```typescript
type PromptCommand = {
  type: 'prompt'
  name: string
  description: string
  source: 'builtin' | 'userSettings' | 'projectSettings' | 'policySettings' | 'bundled' | 'plugin'
  loadedFrom?: 'commands_DEPRECATED' | 'skills' | 'plugin' | 'managed' | 'bundled' | 'mcp'
  contentLength: number
  progressMessage: string
  getPromptForCommand(args: string, context: ToolUseContext): Promise<ContentBlockParam[]>
  
  // Skill-specific fields
  allowedTools?: string[]
  argumentHint?: string
  argNames?: string[]
  whenToUse?: string
  version?: string
  model?: string
  disableModelInvocation?: boolean
  userInvocable?: boolean
  paths?: string[]  // Conditional activation paths
  hooks?: HooksSettings
  skillRoot?: string
  context?: 'inline' | 'fork'
  agent?: string
  effort?: EffortValue
}
```

### LocalCommand

Executes entirely on the client side:

```typescript
type LocalCommand = {
  type: 'local'
  name: string
  description: string
  call(args: string, context: ToolUseContext): Promise<void | string>
}
```

## 5.3 Command Registry

The `commands.ts` file is essentially a massive import manifest. Here's the complete inventory of built-in commands, organized by category:

### Git & Version Control
| Command | Type | Description |
|---------|------|-------------|
| `/commit` | prompt | Generate commit message and commit |
| `/commit-push-pr` | prompt | Commit, push, and create PR |
| `/branch` | local | Branch management |
| `/diff` | local | Show changes in current session |
| `/pr_comments` | prompt | Process PR review comments |
| `/review` | prompt | Code review |
| `/ultrareview` | prompt | Deep code review (variant) |

### Session Management
| Command | Type | Description |
|---------|------|-------------|
| `/clear` | local | Clear conversation |
| `/compact` | local | Compact conversation history |
| `/resume` | local | Resume previous session |
| `/session` | local | Session management |
| `/share` | local | Share conversation transcript |
| `/export` | local | Export conversation |
| `/copy` | local | Copy last response |
| `/rewind` | local | Rewind to previous state |

### Configuration
| Command | Type | Description |
|---------|------|-------------|
| `/config` | local | Configuration settings |
| `/permissions` | local | Permission management |
| `/theme` | local | Theme selection |
| `/model` | local | Model selection |
| `/effort` | local | Effort level setting |
| `/fast` | local | Fast mode toggle |
| `/output-style` | local | Output style selection |
| `/env` | local | Environment variables |
| `/vim` | local | Vim mode toggle |
| `/keybindings` | local | Keybinding configuration |

### Memory & Context
| Command | Type | Description |
|---------|------|-------------|
| `/memory` | local | Memory management |
| `/context` | local | Context information |
| `/files` | local | File context management |
| `/add-dir` | local | Add working directory |

### Diagnostics & Debug
| Command | Type | Description |
|---------|------|-------------|
| `/doctor` | local | Diagnostic checks |
| `/status` | local | Current status |
| `/stats` | local | Usage statistics |
| `/cost` | local | Cost tracking |
| `/usage` | local | Usage details |
| `/version` | local | Version information |
| `/help` | local | Help information |
| `/perf-issue` | local | Performance diagnostics |
| `/heapdump` | local | Memory heap dump |

### IDE & Extensions
| Command | Type | Description |
|---------|------|-------------|
| `/ide` | local | IDE integration status |
| `/desktop` | local | Desktop app handoff |
| `/mobile` | local | Mobile companion |
| `/chrome` | local | Chrome extension |
| `/plugin` | local | Plugin management |
| `/reload-plugins` | local | Reload plugins |
| `/mcp` | local | MCP server management |

### Advanced Features
| Command | Type | Description |
|---------|------|-------------|
| `/tasks` | local | Background task management |
| `/plan` | local | Planning mode |
| `/agents` | local | Agent management |
| `/skills` | local | Skill management |
| `/teleport` | local | Session teleportation |
| `/remote-setup` | local | Remote setup (CCR) |
| `/upgrade` | local | Upgrade Claude Code |

### Feature-Gated Commands
| Command | Gate | Description |
|---------|------|-------------|
| `/voice` | `VOICE_MODE` | Voice mode |
| `/bridge` | `BRIDGE_MODE` | Bridge mode |
| `/proactive` | `PROACTIVE/KAIROS` | Proactive mode |
| `/brief` | `KAIROS/KAIROS_BRIEF` | Brief mode |
| `/assistant` | `KAIROS` | Assistant mode |
| `/force-snip` | `HISTORY_SNIP` | Force history snip |
| `/workflows` | `WORKFLOW_SCRIPTS` | Workflow management |
| `/subscribe-pr` | `KAIROS_GITHUB_WEBHOOKS` | PR subscription |
| `/ultraplan` | `ULTRAPLAN` | Ultra planning mode |
| `/torch` | `TORCH` | Unknown |
| `/peers` | `UDS_INBOX` | Peer management |
| `/fork` | `FORK_SUBAGENT` | Fork subagent |
| `/buddy` | `BUDDY` | Buddy mode |

### Internal (Anthropic Only)
| Command | Description |
|---------|-------------|
| `/ant-trace` | Internal trace analysis |
| `/backfill-sessions` | Session backfill |
| `/break-cache` | Cache invalidation |
| `/mock-limits` | Mock rate limits |
| `/sandbox-toggle` | Sandbox toggle |
| `/security-review` | Security review |
| `/bughunter` | Bug hunting mode |
| `/autofix-pr` | Auto-fix PR issues |
| `/install-github-app` | Install GitHub app |
| `/install-slack-app` | Install Slack app |
| `/btw` | By-the-way system |
| `/good-claude` | Positive feedback |
| `/issue` | Issue reporting |
| `/stickers` | Sticker system |

## 5.4 Command Execution Flow

When a user types `/command args`:

```
1. Input parsed for slash command prefix
2. Command looked up in registry (built-in + skills + plugins)
3. For PromptCommand:
   a. getPromptForCommand(args, context) called
   b. Returns ContentBlockParam[] (text blocks, potentially with images)
   c. Content injected as a user message
   d. Model processes and responds
4. For LocalCommand:
   a. call(args, context) executed directly
   b. Result displayed as local output
   c. No model interaction
```

## 5.5 Skills as Commands

Skills are the extensible command system. They are loaded from multiple sources:

```typescript
// From commands.ts
import { getSkillDirCommands, getDynamicSkills } from './skills/loadSkillsDir.js'
import { getBundledSkills } from './skills/bundledSkills.js'
import { getBuiltinPluginSkillCommands } from './plugins/builtinPlugins.js'
import { getPluginCommands, getPluginSkills } from './utils/plugins/loadPluginCommands.js'
```

Sources in priority order:
1. **Policy settings** — Managed/enterprise-configured skills
2. **User settings** — `~/.claude/skills/`
3. **Project settings** — `.claude/skills/` in project directory
4. **Additional directories** — `--add-dir` paths
5. **Bundled skills** — Ship with Claude Code
6. **Plugin skills** — From installed plugins
7. **MCP skills** — From connected MCP servers
8. **Legacy commands** — `.claude/commands/` (deprecated)

## 5.6 The `/insights` Pattern — Lazy Loading

Some heavy commands use a lazy-loading pattern to avoid import cost:

```typescript
const usageReport: Command = {
  type: 'prompt',
  name: 'insights',
  description: 'Generate a report analyzing your Claude Code sessions',
  contentLength: 0,
  progressMessage: 'analyzing your sessions',
  source: 'builtin',
  async getPromptForCommand(args, context) {
    const real = (await import('./commands/insights.js')).default
    if (real.type !== 'prompt') throw new Error('unreachable')
    return real.getPromptForCommand(args, context)
  },
}
```

The `insights.ts` module is 113KB (3,200 lines) and includes heavy rendering logic. By lazy-loading it, the startup cost is deferred until the command is actually used.

## 5.7 Command Assembly

The `getCommands()` function assembles the full command list:

```typescript
export function getCommands(cwd: string): Command[] {
  return [
    // All built-in commands
    addDir, clear, commit, compact, config, cost, diff, doctor,
    feedback, help, ide, login, logout, memory, model, permissions,
    plan, resume, review, session, share, skills, status, tasks,
    teleport, theme, usage, version, vim,
    // Feature-gated commands
    ...(voiceCommand ? [voiceCommand] : []),
    ...(bridge ? [bridge] : []),
    ...(proactive ? [proactive] : []),
    // ... many more
  ]
}
```

For remote/bridge mode, commands are filtered:
```typescript
export function filterCommandsForRemoteMode(commands: Command[]): Command[] {
  // Remove commands that don't work in remote sessions
}
```

## 5.8 Skill-Tool Integration

Skills bridge the command and tool systems. The `SkillTool` allows the model to invoke any skill:

```typescript
// From tools/SkillTool/SkillTool.ts
// The model can call SkillTool with a skill name to execute it
```

The `getSlashCommandToolSkills()` function provides skills to the QueryEngine so they appear in the system prompt. Skills can include:
- `whenToUse` — Hints for the model about when to invoke the skill
- `allowedTools` — Tools the skill can use
- `model` — Override model for skill execution
- `context: 'fork'` — Execute in a forked context
- `agent` — Execute via a specific agent type

---

*Next: [Chapter 6 — Multi-Agent and Coordination](06-multi-agent-and-coordination.md)*
