# Claude Code Source Analysis

Deep textbook-style analysis of the Claude Code source code leaked on March 31, 2026.

~1,900 TypeScript files, 512,000+ lines. Written using Opus 4.6 with direct codebase access.

## Table of Contents

| # | Chapter | Focus |
|---|---------|-------|
| 1 | [Overview and Leak Context](research/01-overview-and-leak-context.md) | What leaked, how, significance |
| 2 | [Architecture Overview](research/02-architecture-overview.md) | Tech stack, module map, startup |
| 3 | [Tool System](research/03-tool-system.md) | ~40 tools, registry, execution, permissions |
| 4 | [Query Engine](research/04-query-engine.md) | LLM core, streaming, agentic loop |
| 5 | [Command System](research/05-command-system.md) | Slash commands, registry, ~100 commands |
| 6 | [Multi-Agent & Coordination](research/06-multi-agent-and-coordination.md) | AgentTool, coordinator, team swarms |
| 7 | [IDE Bridge & Plugins](research/07-ide-bridge-and-plugins.md) | VS Code, JetBrains, plugin system |
| 8 | [Memory, Skills & Proactive Features](research/08-memory-skills-and-proactive-features.md) | memdir, skills, KAIROS, cron |
| 9 | [Permission & Security Model](research/09-permission-and-security-model.md) | toolPermission, bypassPermissions, trust |
| 10 | [Implications & Comparison](research/10-implications-and-comparison.md) | Anthropic's approach, vs competitors, legal |

## Reading Guide

See [OUTLINE.md](research/OUTLINE.md) for prerequisites and chapter-by-chapter guide.

## Source
Leaked via npm source map in `@anthropic-ai/claude-code` v2.1.88, March 31, 2026.
