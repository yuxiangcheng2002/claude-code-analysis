# Claude Code Source Analysis — Reading Guide

## About This Project

This is a deep textbook-style analysis of the Claude Code source code, which was accidentally leaked on March 31, 2026 via a source map file bundled into the `@anthropic-ai/claude-code` npm package (version 2.1.88).

The leaked codebase contains ~1,900 TypeScript files and over 512,000 lines of code. This analysis examines the architecture, design patterns, and implications of Anthropic's flagship agentic coding tool.

## Prerequisites

- Familiarity with TypeScript and React
- Basic understanding of LLM APIs (tool calls, streaming, system prompts)
- Interest in agentic AI systems and their architecture

## Chapter Guide

### Part I: Context and Architecture

| # | Chapter | Lines | Focus |
|---|---------|-------|-------|
| 1 | [Overview and Leak Context](01-overview-and-leak-context.md) | ~140 | What leaked, how, significance, external coverage |
| 2 | [Architecture Overview](02-architecture-overview.md) | ~290 | Tech stack, module map, execution modes, startup optimization |

### Part II: Core Systems

| # | Chapter | Lines | Focus |
|---|---------|-------|-------|
| 3 | [Tool System](03-tool-system.md) | ~355 | Tool.ts, all ~40 tools, registry, execution flow, deferred loading |
| 4 | [Query Engine](04-query-engine.md) | ~360 | QueryEngine.ts, query.ts, streaming, agentic loop, compaction |
| 5 | [Command System](05-command-system.md) | ~290 | Slash commands, command registry, ~100 commands, skill integration |

### Part III: Advanced Features

| # | Chapter | Lines | Focus |
|---|---------|-------|-------|
| 6 | [Multi-Agent and Coordination](06-multi-agent-and-coordination.md) | ~335 | AgentTool, swarm mode, coordinator mode, system prompt |
| 7 | [IDE Bridge and Plugins](07-ide-bridge-and-plugins.md) | ~340 | Bridge protocol, VS Code/JetBrains, plugins, MCP integration |
| 8 | [Memory, Skills, and Proactive Features](08-memory-skills-and-proactive-features.md) | ~400 | Memory system, skill loading, KAIROS, voice, scheduling |

### Part IV: Security and Implications

| # | Chapter | Lines | Focus |
|---|---------|-------|-------|
| 9 | [Permission and Security Model](09-permission-and-security-model.md) | ~350 | Permission modes, rules, classifier, trust model |
| 10 | [Implications and Comparison](10-implications-and-comparison.md) | ~230 | Industry comparison, design decisions, lessons, roadmap |

## Suggested Reading Paths

### For Software Engineers
Start with Chapters 2 → 3 → 4 → 6 for the core engineering architecture.

### For Security Researchers
Read Chapters 9 → 3 (permission sections) → 6 (agent isolation) → 1 (context).

### For Product Managers
Chapters 1 → 8 (unreleased features) → 10 (comparison and implications).

### For AI Researchers
Chapters 4 (query engine) → 6 (multi-agent) → 8 (memory and proactive) → 10.

### Quick Overview
Chapters 1 → 2 → 10 for the executive summary.

## Key Takeaways

1. **Scale**: ~512K lines of TypeScript — this is not a weekend project. It's a production system with enterprise-grade engineering.

2. **React in the Terminal**: The entire UI is React/Ink, making Claude Code a reactive CLI application with 146+ components.

3. **Everything is a Tool**: 40+ tools with a unified permission, execution, and rendering model.

4. **Multi-Agent First**: Coordinator mode, swarm teams, and agent forking show that Anthropic views multi-agent orchestration as fundamental, not an add-on.

5. **20+ Hidden Features**: KAIROS, BUDDY, voice, browser, workflows — the shipped product is a fraction of what's built.

6. **Permission Defense-in-Depth**: Rules → hooks → classifier → interactive → hooks. Multiple layers that reflect serious thinking about AI safety in practice.

7. **Prompt Engineering as Architecture**: The coordinator system prompt is thousands of words of behavioral specification — prompts are treated as code.
