# Chapter 1: Overview and Leak Context

## 1.1 What Happened

On March 31, 2026, security researcher Chaofan Shou ([@shoucccc](https://twitter.com/shoucccc)) discovered that Anthropic's Claude Code — their flagship agentic CLI tool — had its entire source code exposed through a JavaScript source map file (`cli.js.map`) included in version 2.1.88 of the `@anthropic-ai/claude-code` npm package.

The source map file was approximately **59.8 MB** in size and, when decompressed, yielded the complete TypeScript source code of Claude Code: **approximately 1,900 TypeScript files and over 512,000 lines of code**.

## 1.2 How It Happened

The leak was caused by a source map file being inadvertently bundled into the published npm package. Source map files (`.map` files) are developer tools that map bundled/minified JavaScript back to the original source code — they are never intended for production distribution.

The root cause appears to trace back to a Bun bundler bug. Anthropic acquired the Bun JavaScript runtime at the end of 2025, and Claude Code is built on top of it. A Bun bug report ([oven-sh/bun#28001](https://github.com/oven-sh/bun/issues/28001)), filed on March 11, 2026, documented that source maps were being served in production mode despite Bun's documentation stating they should be disabled. The issue was still open at the time of the leak.

As one commenter on Twitter noted: *"accidentally shipping your source map to npm is the kind of mistake that sounds impossible until you remember that a significant portion of the codebase was probably written by the AI you are shipping."*

### Timeline

| Time | Event |
|------|-------|
| Early morning, Mar 31 | Anthropic publishes `@anthropic-ai/claude-code@2.1.88` to npm |
| ~Hours later | Chaofan Shou discovers the embedded `cli.js.map` file |
| Within hours | Multiple GitHub repos surface with the extracted source |
| Same day | Ars Technica, The Register, VentureBeat, NDTV report on the leak |
| Apr 1 | The Hacker News publishes detailed analysis; Anthropic confirms |

### Not the First Time

As NDTV noted, this was the **second time** Claude Code's source code leaked in a year, suggesting systemic issues in Anthropic's build pipeline and npm publishing workflow.

## 1.3 Scale of the Leak

The leaked codebase is massive by any standard:

| Metric | Value |
|--------|-------|
| Total TypeScript files | ~1,900 |
| Total lines of code | ~512,000+ |
| Source map file size | 59.8 MB |
| Top-level `src/` modules | 37+ directories |
| Tool implementations | ~40+ |
| Slash commands | ~100+ |
| React/Ink UI components | ~146 |
| Utility files | ~330+ |

Key files by size:
- `main.tsx` — 4,683 lines (803 KB) — the CLI entrypoint
- `query.ts` — 1,729 lines — the core LLM query loop
- `QueryEngine.ts` — 1,295 lines — the session/conversation engine
- `Tool.ts` — 792 lines — tool type system
- `commands.ts` — 754 lines — command registry
- `interactiveHelpers.tsx` — 1,570 lines — terminal UI helpers

## 1.4 What Was Revealed

The leak exposed far more than just how Claude Code works at a technical level. It revealed:

### 1.4.1 Feature Flags and Hidden Capabilities

The source contains **44+ compile-time feature flags** (via `feature()` from `bun:bundle`) and numerous GrowthBook runtime feature gates. Many gate unreleased capabilities:

| Feature Flag | What It Gates |
|-------------|---------------|
| `KAIROS` | Persistent autonomous assistant mode |
| `PROACTIVE` | Proactive/background agent capabilities |
| `BUDDY` | Companion agent system |
| `COORDINATOR_MODE` | Multi-agent coordinator architecture |
| `BRIDGE_MODE` | IDE bridge (VS Code, JetBrains) |
| `VOICE_MODE` | Voice input/output mode |
| `HISTORY_SNIP` | Advanced context window management |
| `TEAMMEM` | Team memory sharing system |
| `AGENT_TRIGGERS` | Cron-based agent scheduling |
| `AGENT_TRIGGERS_REMOTE` | Remote agent triggers |
| `DAEMON` | Background daemon process |
| `WEB_BROWSER_TOOL` | Browser automation tool |
| `MONITOR_TOOL` | System monitoring tool |
| `WORKFLOW_SCRIPTS` | Workflow automation system |
| `FORK_SUBAGENT` | Forked subagent execution |
| `UDS_INBOX` | Unix domain socket peer communication |
| `ULTRAPLAN` | Advanced planning system |
| `TORCH` | Unknown internal feature |
| `CONTEXT_COLLAPSE` | Context window compression |
| `TERMINAL_PANEL` | Terminal capture tool |

### 1.4.2 Anti-Distillation Measures

The source revealed that Anthropic employs active anti-distillation techniques — specifically, **fake tool injection** gated behind `tengu_anti_distill_fake_tool_injection`. This inserts fake tools into the system prompt when a first-party CLI session is detected, presumably to detect if competitors are training on Claude Code's outputs.

### 1.4.3 System Prompts

The complete system prompt construction logic is visible, including how CLAUDE.md files are loaded, how memory prompts are structured, and how context is assembled for each query.

### 1.4.4 Internal Analytics

Extensive use of analytics events (`logEvent`) with the `tengu_` prefix (Anthropic's internal codename) reveals detailed telemetry about user behavior, tool usage, permission decisions, and more.

## 1.5 Significance

### For Developers and Users

The leak provides unprecedented transparency into how a commercial AI coding tool actually works. Developers can now understand:
- Exactly what system prompts Claude Code uses
- How the permission model works and where trust boundaries exist
- What data is collected and sent to Anthropic
- How features like auto-compaction, memory, and multi-agent coordination function

### For Competitors

The source code provides a detailed blueprint for building agentic coding tools. Key architectural decisions — the tool-call loop, permission model, context management strategy, and multi-agent coordination pattern — are now public knowledge.

### For Security Researchers

The permission system, sandbox model, and trust boundaries are fully visible, enabling thorough security audits. The classifier-based auto-approval system for bash commands, the denial tracking state machine, and the hook-based permission override system can all be studied in detail.

### For the AI Industry

The leak reveals Anthropic's strategic direction: autonomous agents (KAIROS), persistent assistant mode, proactive behavior, team coordination, voice interfaces, and browser automation. These represent the likely roadmap for Claude Code over the coming months.

## 1.6 Legal and Ethical Context

The source code is proprietary to Anthropic. While npm packages are publicly downloadable and source maps are technically part of the published artifact, the intent was clearly not to distribute the source code. Multiple GitHub repositories hosting the extracted source have been subject to DMCA takedown requests.

This analysis is conducted for research and educational purposes, examining the architecture and design patterns of a significant commercial AI system. All code snippets are quoted for commentary and analysis purposes.

## 1.7 External Coverage

Key external analyses and reports:

- **Ars Technica**: [Entire Claude Code CLI source code leaks thanks to exposed map file](https://arstechnica.com/ai/2026/03/entire-claude-code-cli-source-code-leaks-thanks-to-exposed-map-file/)
- **The Register**: [Anthropic accidentally exposes Claude Code source code](https://www.theregister.com/2026/03/31/anthropic_claude_code_source_code/)
- **VentureBeat**: [Claude Code's source code appears to have leaked](https://venturebeat.com/technology/claude-codes-source-code-appears-to-have-leaked-heres-what-we-know)
- **The Hacker News**: [Claude Code Source Leaked via npm Packaging Error](https://thehackernews.com/2026/04/claude-code-tleaked-via-npm-packaging.html)
- **Alex Kim's Blog**: [The Claude Code Source Leak: fake tools, frustration regexes, undercover mode, and more](https://alex000kim.com/posts/2026-03-31-claude-code-source-leak/)
- **Engineer's Codex**: [Diving into Claude Code's Source Code Leak](https://read.engineerscodex.com/p/diving-into-claude-codes-source-code)
- **Sathwick's Blog**: [Reverse-Engineering Claude Code: A Deep Dive](https://sathwick.xyz/blog/claude-code.html)
- **WaveSpeed AI**: [Claude Code Leaked Source: BUDDY, KAIROS & Every Hidden Feature](https://wavespeed.ai/blog/posts/claude-code-leaked-source-hidden-features/)
- **ClaudeFast**: [Claude Code Source Leak: Everything Found (2026)](https://claudefa.st/blog/guide/mechanics/claude-code-source-leak)

---

*Next: [Chapter 2 — Architecture Overview](02-architecture-overview.md)*
