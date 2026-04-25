# Claude Code Source Analysis

Deep textbook-style analysis of the Claude Code source code leaked on March 31, 2026.

~1,900 TypeScript files, 512,000+ lines of TypeScript/React. Written using Opus 4.6 with direct source code access.

## Background

On March 31, 2026, security researcher Chaofan Shou discovered that Anthropic's Claude Code — their flagship agentic CLI tool — had its entire source code exposed through a 59.8 MB JavaScript source map file (`cli.js.map`) bundled into `@anthropic-ai/claude-code@2.1.88` on npm. The root cause was a Bun bundler bug that shipped source maps in production mode.

## Table of Contents

| # | Chapter | Lines | Focus |
|---|---------|-------|-------|
| 1 | [Overview and Leak Context](research/01-overview-and-leak-context.md) | 141 | What leaked, how, timeline, significance, external coverage |
| 2 | [Architecture Overview](research/02-architecture-overview.md) | 292 | Tech stack (Bun/React/Ink), module map, execution modes, startup optimization |
| 3 | [Tool System](research/03-tool-system.md) | 355 | Tool.ts interface, all ~40+ tools, registry, execution flow, deferred loading |
| 4 | [Query Engine](research/04-query-engine.md) | 357 | QueryEngine.ts + query.ts, streaming, agentic loop, auto-compaction, retry |
| 5 | [Command System](research/05-command-system.md) | 291 | Slash commands, command registry, ~100 commands, skill-tool integration |
| 6 | [Multi-Agent & Coordination](research/06-multi-agent-and-coordination.md) | 334 | AgentTool, TeamCreateTool, coordinator mode, swarm system prompt |
| 7 | [IDE Bridge & Plugins](research/07-ide-bridge-and-plugins.md) | 343 | Bridge protocol, VS Code/JetBrains, plugins, MCP integration, hooks |
| 8 | [Memory, Skills & Proactive Features](research/08-memory-skills-and-proactive-features.md) | 401 | memdir/, skills/, KAIROS, PROACTIVE, voice, cron scheduling |
| 9 | [Permission & Security Model](research/09-permission-and-security-model.md) | 351 | Permission modes, rules, bash classifier, denial tracking, trust model |
| 10 | [Implications & Comparison](research/10-implications-and-comparison.md) | 227 | vs Copilot/Cursor/Aider/Codex, architectural innovations, roadmap |

**Total: ~3,092 lines across 10 chapters**

## Reading Paths

| Audience | Recommended Path |
|----------|-----------------|
| Software Engineers | 2 → 3 → 4 → 6 |
| Security Researchers | 9 → 3 → 6 → 1 |
| Product Managers | 1 → 8 → 10 |
| AI Researchers | 4 → 6 → 8 → 10 |
| Quick Overview | 1 → 2 → 10 |

## Key Findings

- **44+ compile-time feature flags** via `bun:bundle` — many gating unreleased features
- **20+ hidden features**: KAIROS (autonomous assistant), BUDDY, voice mode, browser tool, workflow scripts
- **React/Ink terminal UI** with 146+ components — a fully reactive CLI application
- **Multi-layer permission system**: rules → hooks → ML classifier → interactive prompt → hooks
- **Coordinator mode**: Full multi-agent orchestration with detailed behavioral prompts
- **Anti-distillation measures**: Fake tool injection to detect training data contamination

## Reading Guide

See [OUTLINE.md](research/OUTLINE.md) for prerequisites and a chapter-by-chapter guide.

## Source

Leaked via npm source map in `@anthropic-ai/claude-code` v2.1.88, March 31, 2026.
Discovered by security researcher Chaofan Shou ([@shoucccc](https://twitter.com/shoucccc)).
