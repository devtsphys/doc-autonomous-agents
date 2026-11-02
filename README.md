# Autonomous AI Agents: Reference Card & Cheat Sheet

> **Edition:** November 2026 · **Scope:** Claude Code, OpenCode, Agent Skills, n8n, Hermes Agent, OpenClaw, plus the surrounding ecosystem (MCP, Codex CLI, Gemini CLI, Aider, LangGraph, CrewAI, AutoGen, OpenAI Agents SDK, Claude Agent SDK, Cursor, etc.)
> **Caveat:** This field changes monthly. Command names, flags, defaults and version numbers below are accurate to the best of my knowledge but **always verify against the official docs (`--help`, `/help`, the project README)** before relying on them in production. Items marked ⚠️ are the most volatile.

---

## Table of Contents

1. [Core Concepts](#1-core-concepts)
2. [Universal Architecture Patterns](#2-universal-architecture-patterns)
3. [Landscape at a Glance](#3-landscape-at-a-glance)
4. [Claude Code](#4-claude-code)
5. [Agent Skills (open SKILL.md standard)](#5-agent-skills)
6. [MCP: Model Context Protocol](#6-mcp-model-context-protocol)
7. [OpenCode](#7-opencode)
8. [n8n](#8-n8n)
9. [Hermes Agent (Nous Research)](#9-hermes-agent-nous-research)
10. [OpenClaw](#10-openclaw)
11. [Other Tools Worth Knowing](#11-other-tools-worth-knowing)
12. [Comparison Matrix](#12-comparison-matrix)
13. [Decision Guide: Which Tool When?](#13-decision-guide-which-tool-when)
14. [Best Practices (Universal)](#14-best-practices-universal)
15. [Prompting & Context Engineering Cheat Sheet](#15-prompting--context-engineering)
16. [Security & Safety Checklist](#16-security--safety-checklist)
17. [Typical Use Cases & Recipes](#17-typical-use-cases--recipes)
18. [Failure Modes & Troubleshooting](#18-failure-modes--troubleshooting)
19. [Cost & Performance Tuning](#19-cost--performance-tuning)
20. [Evaluation & Observability](#20-evaluation--observability)
21. [Glossary](#21-glossary)
22. [Quick-Reference One-Pagers](#22-quick-reference-one-pagers)

---

## 1. Core Concepts

| Term | Definition |
|---|---|
| **LLM** | Stateless next-token predictor. Takes a prompt, returns text (or a structured tool call). |
| **Agent** | An LLM running in a **loop**: *observe → think → act (tool call) → observe result → repeat* until a goal is met or a stop condition hits. |
| **Autonomous agent** | An agent that decides its own next steps with little or no human approval per step. Autonomy is a **dial**, not a switch. |
| **Workflow** | A *predefined* graph of steps (possibly with LLM nodes). The developer controls the path; the LLM fills in steps. |
| **Harness / scaffold** | Everything around the model: system prompt, tool definitions, permission system, context management, memory, loop control. *Claude Code, OpenCode, Hermes, OpenClaw are harnesses.* |
| **Tool** | A function the model may call (read file, run shell, HTTP request, search). Exposed via JSON schema. |
| **Context window** | The model's working memory (prompt + history + tool results). Finite and the #1 resource to manage. |
| **Memory** | Persistence beyond a single context window: files, DBs, vector stores, summaries. |
| **Skill** | A packaged, reusable bundle of instructions (+ scripts/resources) that an agent loads **on demand**. |
| **Subagent** | A child agent with its own context window, spawned by a parent for a focused task; returns a summary. |
| **MCP** | Model Context Protocol: open standard to connect agents to external tools, data and prompts. |
| **Hooks** | Deterministic scripts triggered by agent lifecycle events (before/after tool use, on stop, etc.). |
| **Heartbeat / cron** | Time-triggered agent runs, which make an agent *proactive* instead of purely reactive. |
| **Human-in-the-loop (HITL)** | Approval gates at risky steps. |
| **Sandbox** | Isolated execution environment (container, VM, restricted FS/network) limiting blast radius. |

### Autonomy Spectrum

```
Level 0  Chat                → human copy/pastes everything
Level 1  Autocomplete        → inline suggestions (Copilot-style)
Level 2  Assisted agent      → agent proposes; human approves every action
Level 3  Supervised agent    → auto-approves safe actions, asks for risky ones   ← typical sweet spot
Level 4  Autonomous agent    → runs for minutes/hours; human reviews outcome
Level 5  Always-on agent     → persistent, proactive (cron, heartbeat, messaging), long-lived memory
```

### Agent vs. Workflow: When to Choose Which

| Criterion | Prefer **Workflow** (e.g. n8n flow) | Prefer **Agent** (e.g. Claude Code) |
|---|---|---|
| Steps known in advance | ✅ | ❌ |
| Needs determinism / audit trail | ✅ | ⚠️ add hooks/logs |
| Open-ended, exploratory task | ❌ | ✅ |
| Cost predictability | ✅ | ⚠️ variable |
| Many heterogeneous SaaS integrations | ✅ | ⚠️ via MCP |
| Needs to read/edit a codebase | ❌ | ✅ |
| **Best of both** | Workflow *calls* an agent node for the fuzzy step, or agent *calls* a workflow via webhook/MCP | |

---

## 2. Universal Architecture Patterns

### 2.1 The Core Agent Loop

```
        ┌────────────────────────────────────────────────┐
        │                  USER / TRIGGER                │
        │  (prompt, cron, webhook, chat message, event)  │
        └───────────────────────┬────────────────────────┘
                                ▼
┌──────────────────────────────────────────────────────────────┐
│                          HARNESS                              │
│  ┌──────────────┐  ┌───────────────┐  ┌──────────────────┐   │
│  │System prompt │  │ Context mgmt  │  │ Permission system│   │
│  │ + rules file │  │ (compaction)  │  │ (allow/ask/deny) │   │
│  └──────┬───────┘  └───────┬───────┘  └────────┬─────────┘   │
│         ▼                  ▼                   ▼             │
│   ┌──────────────────────────────────────────────────┐       │
│   │                     LLM CALL                     │◄──┐   │
│   └───────────────────────┬──────────────────────────┘   │   │
│                           ▼                              │   │
│              text answer?  ──yes──► DONE / ask human     │   │
│                  │ no (tool_use)                         │   │
│                  ▼                                       │   │
│   ┌──────────────────────────────────────────────────┐   │   │
│   │ TOOL EXECUTION (built-in, MCP, skills, subagents)│───┘   │
│   └──────────────────────────────────────────────────┘ result│
│                                                              │
│   Memory (files / DB / vectors)   Hooks   Logs / traces      │
└──────────────────────────────────────────────────────────────┘
```

### 2.2 Building Blocks Every Agent Harness Has

| Block | Purpose | Examples |
|---|---|---|
| **Instruction file** | Persistent project/persona rules loaded every session | `CLAUDE.md`, `AGENTS.md`, `SOUL.md`, `.cursorrules` |
| **Tools** | Act on the world | Bash, Read/Edit/Write, Grep/Glob, WebFetch, MCP tools |
| **Permissions** | Constrain tools | allow/ask/deny rules, modes (plan / accept-edits), sandbox |
| **Skills** | On-demand expertise (progressive disclosure) | `SKILL.md` folders |
| **Subagents** | Context isolation + parallelism | Claude Code subagents, OpenCode agents, OpenClaw spawn |
| **Memory** | Cross-session knowledge | Markdown memory files, SQLite, vector DB |
| **Hooks / events** | Deterministic guardrails & automation | `PreToolUse`, `PostToolUse`, `Stop` |
| **Scheduler** | Proactivity | cron, heartbeat, n8n triggers |
| **Channels** | Where humans talk to it | CLI, IDE, Telegram, Slack, WhatsApp, webhook |
| **Model router** | Pick model per task/cost | provider config, aliases, fallbacks |

### 2.3 Multi-Agent Topologies

| Pattern | Shape | When | Risk |
|---|---|---|---|
| **Single agent + tools** | 1 loop | Most tasks. **Start here.** | Context bloat |
| **Orchestrator–workers** | Parent delegates to subagents, merges results | Parallel research, large refactors | Coordination overhead, cost ×N |
| **Pipeline / chain** | A → B → C | Content pipeline (research → draft → edit) | Error propagation |
| **Router** | Classifier picks specialist | Support triage | Misrouting |
| **Evaluator–optimizer** | Generator + critic loop | Code review, writing quality | Infinite loops: cap iterations |
| **Swarm / peer network** | Agents message each other (A2A) | Cross-vendor interop | Hard to debug and secure |
| **Human-in-the-loop gate** | Agent pauses for approval | Payments, prod deploys, outbound email | Latency |

> **Rule of thumb:** Add agents only when a single agent demonstrably fails. Every extra agent multiplies tokens, latency and failure surface.

### 2.4 Memory Architecture Types

| Type | Mechanism | Pros | Cons |
|---|---|---|---|
| **In-context** | Everything in the prompt | Simple, precise | Limited, expensive |
| **Compaction / summarization** | Summarize old turns | Extends sessions | Lossy |
| **File-based (markdown)** | `MEMORY.md`, daily logs, notes | Human-readable, versionable, editable | Needs curation, grows unbounded |
| **Structured DB (SQLite/Postgres)** | Session transcripts, facts | Queryable | Schema upkeep |
| **Vector / semantic** | Embeddings + retrieval | Fuzzy recall at scale | Retrieval misses, noise |
| **Skill-as-memory** | Agent writes reusable procedures into skills | Procedural learning | Quality drift, security |
| **Knowledge graph** | Entities + relations | Reasoning over relations | Heavy to build |

---

## 3. Landscape at a Glance

| Tool | Category | Primary interface | Models | License / Model | Self-host |
|---|---|---|---|---|---|
| **Claude Code** | Coding agent harness | Terminal, IDE ext., desktop, web, mobile | Claude (Anthropic API, Bedrock, Vertex, etc.) | Proprietary (Anthropic) | Runs locally; model via API |
| **OpenCode** | Coding agent harness (TUI) | Terminal TUI, desktop, IDE | 75+ providers (Claude, GPT, Gemini, local) | Open source | ✅ |
| **Agent Skills** | **Format/standard** for reusable capabilities | Folders of `SKILL.md` | Model-agnostic in principle | Open spec (originated at Anthropic) | n/a |
| **n8n** | Workflow automation + AI agent nodes | Visual web editor | Any (OpenAI, Anthropic, Ollama…) | Fair-code (Sustainable Use License) | ✅ |
| **Hermes Agent** | Self-improving persistent agent | CLI + messaging gateways | Model-agnostic (OpenRouter, Nous Portal, local…) | MIT | ✅ |
| **OpenClaw** | Self-hosted personal assistant (formerly Clawdbot/Moltbot) | Messaging apps via Gateway daemon | Multi-provider | Open source | ✅ |
| **MCP** | **Protocol** for tools/data | n/a | n/a | Open standard | n/a |

---

## 4. Claude Code

**What it is:** Anthropic's agentic coding tool. Lives in your terminal (also VS Code/JetBrains extensions, desktop app, web, mobile remote). Reads your codebase, edits files, runs commands, handles git, and can be extended with MCP, skills, hooks, subagents and plugins. Also usable headlessly (`-p`) in CI and through the **Claude Agent SDK**.

### 4.1 Architecture

```
┌───────────────────────── Claude Code ─────────────────────────┐
│  Interfaces: CLI · IDE · Desktop · Web · Slack · GitHub Action│
├───────────────────────────────────────────────────────────────┤
│  Memory:   CLAUDE.md (enterprise → user → project → local)    │
│            auto-memory · @imports                             │
├───────────────────────────────────────────────────────────────┤
│  Agent loop (Claude model) ⇄ Built-in tools                   │
│     Read · Edit · Write · Glob · Grep · Bash · WebFetch ·     │
│     WebSearch · Task(subagent) · TodoWrite · NotebookEdit     │
├───────────────────────────────────────────────────────────────┤
│  Extensions:  MCP servers · Skills · Slash commands ·         │
│               Subagents · Hooks · Plugins (marketplaces)      │
├───────────────────────────────────────────────────────────────┤
│  Safety: permission rules (allow/ask/deny) · modes ·          │
│          sandboxing · checkpoints/rewind                      │
└───────────────────────────────────────────────────────────────┘
```

### 4.2 Install & Start

```bash
# Native installer (recommended; macOS/Linux/WSL)
curl -fsSL https://claude.ai/install.sh | bash
# Alternative (⚠️ verify current method in docs)
npm install -g @anthropic-ai/claude-code

cd my-project
claude                     # start interactive session
claude "explain this repo" # start with an initial prompt
claude -c                  # continue most recent conversation
claude -r                  # resume: pick a past session
claude -p "query"          # print mode (headless), then exit
cat logs.txt | claude -p "summarize errors"   # pipe input
claude update              # update
claude doctor              # diagnose install
```

### 4.3 Key CLI Flags ⚠️

| Flag | Purpose |
|---|---|
| `-p, --print` | Non-interactive, print result and exit |
| `-c, --continue` / `-r, --resume [id]` | Continue / resume session |
| `--model <alias\|name>` | Pick model (`sonnet`, `opus`, `haiku`, or full ID) |
| `--permission-mode <mode>` | `default`, `acceptEdits`, `plan`, `bypassPermissions` |
| `--allowedTools` / `--disallowedTools` | Pre-approve / block tools, e.g. `"Bash(git log:*)" "Read"` |
| `--output-format <text\|json\|stream-json>` | Machine-readable output for scripts/CI |
| `--max-turns <n>` | Cap agent iterations (headless) |
| `--add-dir <path>` | Give access to extra directories |
| `--mcp-config <file>` | Load MCP servers from JSON |
| `--append-system-prompt "…"` | Add to the system prompt |
| `--dangerously-skip-permissions` | **No approvals.** Only in disposable sandboxes/containers |
| `--verbose` / `--debug` | Troubleshooting |

### 4.4 In-Session Slash Commands ⚠️

| Command | Purpose |
|---|---|
| `/help` | List commands |
| `/init` | Generate a starter `CLAUDE.md` from your repo |
| `/memory` | Edit memory files |
| `/clear` | Reset context (start fresh task, free tokens) |
| `/compact [focus]` | Summarize history to free context |
| `/context` | Visualize context usage |
| `/cost` / `/usage` | Token/cost or plan usage |
| `/model` | Switch model |
| `/permissions` | View/edit allow/ask/deny rules |
| `/agents` | Manage subagents |
| `/mcp` | Manage MCP servers / auth |
| `/hooks` | Configure hooks |
| `/review` | Review a PR/changes |
| `/resume` | Resume earlier session |
| `/rewind` (or Esc Esc) | Roll back conversation/code checkpoints |
| `/status`, `/doctor` | Status & diagnostics |
| `/terminal-setup`, `/vim` | UX setup |
| `/login`, `/logout` | Auth |

**Keyboard:** `Shift+Tab` cycles permission modes (default → auto-accept edits → plan) · `Esc` interrupts · `Esc Esc` rewind · `#`/memory shortcuts · `!cmd` runs bash directly · `@path` references files · `Ctrl+C` cancel/exit.

### 4.5 Memory: `CLAUDE.md`

Loaded automatically at session start (hierarchy: enterprise policy → user `~/.claude/CLAUDE.md` → project `./CLAUDE.md` → local overrides, plus nested folders on demand). Supports `@path/to/file` imports.

```markdown
# CLAUDE.md
## Project
Next.js 15 + TypeScript + Postgres monorepo (pnpm workspaces).

## Commands
- Install: `pnpm install`
- Test (single file): `pnpm vitest run path/to/file.test.ts`
- Lint+typecheck: `pnpm lint && pnpm typecheck`
- Dev: `pnpm dev`

## Conventions
- Functional React components, no default exports
- Use zod for all external input validation
- Never edit files in `/generated`

## Workflow
- Make a plan first for changes touching >3 files
- Run lint + the relevant tests before declaring done
- Commit messages: Conventional Commits

@docs/architecture.md
```

**CLAUDE.md best practices:** keep it short (aim for < ~200 lines), concrete and verifiable; put *commands and gotchas*, not essays; use emphasis ("IMPORTANT") sparingly; prune regularly; move rarely-needed procedures into **skills**.

### 4.6 Settings & Permissions

Files (precedence high → low): enterprise managed → CLI args → `.claude/settings.local.json` (git-ignored) → `.claude/settings.json` (shared) → `~/.claude/settings.json` (user).

```jsonc
// .claude/settings.json
{
  "model": "sonnet",
  "permissions": {
    "allow": [
      "Bash(pnpm test:*)",
      "Bash(pnpm lint)",
      "Bash(git diff:*)",
      "Read(src/**)"
    ],
    "ask":  ["Bash(git push:*)"],
    "deny": [
      "Read(./.env)",
      "Read(./secrets/**)",
      "Bash(rm -rf:*)",
      "Bash(curl:*)"
    ]
  },
  "hooks": {
    "PostToolUse": [
      { "matcher": "Edit|Write",
        "hooks": [{ "type": "command", "command": "pnpm prettier --write \"$CLAUDE_FILE_PATHS\"" }] }
    ]
  },
  "env": { "NODE_ENV": "development" }
}
```

### 4.7 Hooks (deterministic control)

| Event | Fires | Typical use |
|---|---|---|
| `SessionStart` | session begins | inject context (git status, ticket) |
| `UserPromptSubmit` | before prompt processed | add context / block |
| `PreToolUse` | before a tool runs | **block dangerous commands**, validate |
| `PostToolUse` | after a tool succeeds | auto-format, lint, run tests |
| `Notification` | agent needs attention | desktop/Slack ping |
| `Stop` / `SubagentStop` | agent finishes | final checks, notify |
| `PreCompact` | before compaction | save state |

A hook receives JSON on stdin; exit code `0` = ok, `2` = **block** and feed stderr back to the model. *Use hooks (not prompts) for anything that must ALWAYS happen.*

### 4.8 Subagents

Markdown files with YAML frontmatter in `.claude/agents/` (project) or `~/.claude/agents/` (user):

```markdown
---
name: code-reviewer
description: Expert code reviewer. Use PROACTIVELY after any code change.
tools: Read, Grep, Glob, Bash
model: sonnet
---
You are a senior reviewer. Check: correctness, security (OWASP), performance,
readability, test coverage. Output a prioritized list: Critical / Warning / Suggestion.
```

Why: **isolated context** (verbose exploration doesn't pollute main thread), **parallelism**, **tool restriction**, **model choice per task** (cheap model for search, strong for design).

### 4.9 Custom Slash Commands

`.claude/commands/fix-issue.md` → `/fix-issue 123`

```markdown
---
description: Fix a GitHub issue end to end
argument-hint: [issue-number]
allowed-tools: Bash(gh:*), Read, Edit
---
Fix GitHub issue #$ARGUMENTS:
1. `gh issue view $ARGUMENTS`
2. Find relevant code, write a failing test first
3. Implement the fix, run tests
4. Open a PR with `gh pr create`
```

(Skills and slash commands have been converging; a skill can also be invoked explicitly. ⚠️ verify current behavior.)

### 4.10 MCP in Claude Code

```bash
claude mcp add github --transport http https://api.githubcopilot.com/mcp/   # remote (HTTP)
claude mcp add --transport stdio fs -- npx -y @modelcontextprotocol/server-filesystem ~/proj
claude mcp list
claude mcp get github
claude mcp remove github
# scopes: --scope local (default) | project (.mcp.json, shareable) | user
```

### 4.11 Headless / CI / SDK

```bash
# GitHub Actions style
claude -p "Review this diff for security issues" \
  --output-format json --max-turns 5 \
  --allowedTools "Read" "Grep" "Bash(git diff:*)"

# Fan-out over files
for f in $(git diff --name-only main); do
  claude -p "Add missing docstrings to $f" --allowedTools "Edit" "Read"
done
```

**Claude Agent SDK** (Python / TypeScript): the same harness as a library: build custom agents with the same tools, context management, permissions, hooks, subagents and MCP.

### 4.12 Proven Workflows

| Workflow | Steps |
|---|---|
| **Explore → Plan → Code → Commit** | 1) ask it to *read* relevant files, no coding 2) `Shift+Tab` into **plan mode**, get a plan 3) approve & implement 4) tests 5) commit/PR |
| **TDD loop** | write failing tests → confirm fail → commit tests → implement until green (don't modify tests) |
| **Visual iteration** | give screenshot/mock → implement → screenshot result (via Playwright MCP) → iterate 2-3× |
| **Parallel sessions** | `git worktree add ../feat-a` per task; run one `claude` in each |
| **Writer/Reviewer** | one session writes, a fresh session (clean context) reviews |
| **Codebase Q&A / onboarding** | "How does auth work? Trace a login request." |
| **Headless batch** | scripted `claude -p` over many files with tight `--allowedTools` |

### 4.13 Pros / Cons

| ✅ Pros | ❌ Cons |
|---|---|
| Very strong coding quality & agentic reliability | Proprietary harness; tied to Anthropic models (or its cloud partners) |
| Rich extensibility (MCP, skills, hooks, subagents, plugins) | Cost can escalate on long autonomous runs |
| Excellent safety controls (permission rules, plan mode, sandbox, rewind) | Context fills fast on big repos → needs discipline (`/clear`, `/compact`) |
| Headless mode + SDK for CI/automation | Not a general life/personal assistant (no built-in messaging presence) |
| Deep IDE/GitHub/Slack integration | Rate limits on subscription plans |
| Active ecosystem of plugins/marketplaces | Fast-changing features (docs drift) |

### 4.14 Typical Use Cases
Feature implementation · bug hunting from stack traces · large refactors/migrations · test generation · code review · documentation · PR/issue automation · onboarding to unknown codebases · data/script one-offs · DevOps/infra-as-code edits · CI auto-fix bots.

---

## 5. Agent Skills

**What it is:** An **open, file-based format** for packaging procedural know-how for agents. A skill is a **folder** containing a `SKILL.md` (YAML frontmatter + Markdown instructions) and optional scripts, templates and reference files. Originated at Anthropic, and adopted across tools (Claude.ai, Claude Code, Claude API, Agent SDK; compatible patterns in OpenClaw, Hermes Agent, OpenCode and others ⚠️ check each tool's compatibility notes).

### 5.1 Structure

```
my-skill/
├── SKILL.md            ← required: frontmatter + instructions
├── reference/
│   ├── api.md          ← loaded only when needed
│   └── style-guide.md
├── scripts/
│   └── validate.py     ← executed, not loaded into context
└── templates/
    └── report.docx
```

```markdown
---
name: pdf-report-builder
description: Builds branded PDF reports from CSV data. Use when the user asks for a
  report, PDF export, or formatted summary of tabular data.
---
# PDF Report Builder
## Quick start
1. Read the CSV with pandas; validate columns against `reference/schema.md`
2. Run `python scripts/build_report.py --in data.csv --out report.pdf`
3. Verify page count and render page 1 to PNG to visually check

## Rules
- Always use the template in `templates/`
- Never include raw PII columns (see reference/schema.md)

## Advanced
For charts see [reference/charts.md](reference/charts.md)
```

### 5.2 Progressive Disclosure (the key idea)

| Level | What loads | When | Token cost |
|---|---|---|---|
| **1. Metadata** | `name` + `description` only | Always (startup) | ~100 tokens/skill |
| **2. Instructions** | Body of `SKILL.md` | When the model decides the skill is relevant | Small (<5k recommended) |
| **3. Resources** | Linked files / script outputs | Only when referenced | Unlimited, on demand |

→ You can install **dozens of skills** without bloating context. Scripts run via the shell and only their *output* enters context.

### 5.3 Where Skills Live

| Tool | Location (typical ⚠️) |
|---|---|
| Claude Code | `~/.claude/skills/<name>/` (personal), `.claude/skills/<name>/` (project), plugins |
| Claude.ai / API | Uploaded via Settings / Skills API |
| OpenClaw | workspace `skills/`, `~/.openclaw/skills`, bundled; **6-level precedence** (workspace wins) |
| Hermes Agent | Ships with built-in skills; agent can **author new skills from experience** |

### 5.4 Writing Great Skills: Best Practices

1. **Description is everything.** It's the trigger. Say *what it does* **and** *when to use it*, in third person, with concrete keywords users would say.
2. **Be concise.** Claude is already smart; only add what it doesn't know (your conventions, tricky APIs, gotchas).
3. **Match freedom to fragility:** high freedom (text guidance) for flexible tasks; low freedom (exact script) for fragile/destructive ones.
4. **Keep `SKILL.md` < ~500 lines;** push detail into one-level-deep reference files.
5. **Prefer scripts for deterministic work** (validation, conversion): reliable and token-cheap.
6. **Include a checklist/verification step** (e.g. "render and inspect output").
7. **Test with real tasks** across models; iterate from observed failures (build evals *before* writing extensive docs).
8. **One skill = one capability.** Avoid mega-skills with vague descriptions.
9. **Audit third-party skills** like you'd audit code: they can contain scripts and prompt injection. (See §16.)

### 5.5 Skills vs. Other Extension Mechanisms

| | **Skill** | **MCP server** | **Slash command** | **Subagent** | **Hook** | **CLAUDE.md** |
|---|---|---|---|---|---|---|
| Nature | Instructions + scripts | Live tool/data connection | Saved prompt | Separate agent context | Deterministic script | Always-on rules |
| Loaded | On demand | Tool defs at start | When invoked | When delegated | On event | Always |
| Best for | Repeatable procedures, domain expertise | External systems (DBs, SaaS APIs) | Quick shortcuts | Isolation/parallelism | Guarantees | Project conventions |
| Model-decided? | ✅ | ✅ | ❌ (user) | ✅ | ❌ (event) | n/a |

**Pros:** portable, versionable (git), cheap in context, shareable, human-readable. **Cons:** quality depends on description; can't enforce behavior (use hooks); supply-chain/prompt-injection risk; cross-tool compatibility isn't perfect.

---

## 6. MCP: Model Context Protocol

**What:** An open protocol (JSON-RPC 2.0) standardizing how AI apps (clients/hosts) connect to **servers** exposing:
- **Tools**: model-invoked functions
- **Resources**: readable data/context
- **Prompts**: reusable templates
(+ client features such as sampling and elicitation ⚠️)

```
 Host app (Claude Code / OpenCode / n8n / Hermes…)
   └─ MCP client ──(stdio | streamable HTTP)──► MCP server ──► GitHub / DB / Slack / FS / browser…
```

| Transport | Use |
|---|---|
| **stdio** | Local subprocess; simplest; per-user |
| **Streamable HTTP** (successor to SSE) | Remote/hosted servers; OAuth supported |

**Popular servers:** GitHub, filesystem, Postgres/SQLite, Playwright/Puppeteer (browser), Slack, Notion, Linear/Jira, Sentry, Context7 (docs), Brave/Tavily search, Google Drive/Gmail/Calendar, Stripe, Cloudflare.

**Best practices:** enable only servers you need (each adds tool definitions to context) · prefer read-only scopes · pin versions · review server source · use OAuth over long-lived keys · treat tool *outputs* as untrusted input · watch for tool-name collisions.

**Pros:** write once, use everywhere; huge ecosystem. **Cons:** context bloat from many tools; security surface (tool poisoning, confused-deputy); quality of third-party servers varies; auth UX still maturing.

---

## 7. OpenCode

**What it is:** An open-source, provider-agnostic AI coding agent with a polished terminal UI (also desktop app and IDE integration). Positioned as the open alternative to Claude Code. Client/server architecture (the TUI is just one client), so it can be driven remotely or programmatically. (Note: there are two projects named "opencode" historically; the widely used one is **sst/opencode → opencode.ai**. ⚠️ verify.)

### 7.1 Architecture

```
┌──────────── Clients ────────────┐
│ TUI · Desktop · IDE · Web · SDK │
└──────────────┬──────────────────┘
               ▼   HTTP API
┌──────────── opencode server ────────────────────────┐
│ Sessions · Agents (build / plan / custom)           │
│ Tools: read/edit/write/bash/grep/glob/webfetch/...  │
│ LSP integration (diagnostics → model)               │
│ MCP client · Custom tools · Plugins                 │
│ Provider layer (Models.dev: 75+ providers)          │
└─────────────────────────────────────────────────────┘
```

### 7.2 Install & Run

```bash
curl -fsSL https://opencode.ai/install | bash
# or: npm i -g opencode-ai   |  brew install sst/tap/opencode  (⚠️ verify)

cd my-project
opencode                      # launch TUI
opencode run "fix the failing test"   # non-interactive
opencode auth login           # configure provider credentials
opencode models               # list available models
opencode serve                # headless server
opencode upgrade
```

### 7.3 TUI Essentials ⚠️

| Action | How |
|---|---|
| Init project rules | `/init` → generates `AGENTS.md` |
| Switch agent (Build ↔ Plan) | `Tab` |
| Pick model | `/models` |
| New / list sessions | `/new` · `/sessions` |
| Undo / redo file changes | `/undo` · `/redo` |
| Share session link | `/share` |
| Compact context | `/compact` |
| Reference files | `@filename` |
| Run shell | `!command` |
| External editor | `/editor` |

### 7.4 Built-in Agents

- **build**: default, full tool access (edit, bash).
- **plan**: read-only analysis/planning; asks before changes.
- **general** (subagent): complex search/multi-step research, invoked with `@general`.
- **Custom agents**: Markdown files in `.opencode/agent/` (or JSON config) with own prompt, model, tools, permissions.

### 7.3 Config (`opencode.json`)

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "model": "anthropic/claude-sonnet-4-5",      // provider/model
  "small_model": "anthropic/claude-haiku-4-5", // cheap tasks (titles etc.)
  "permission": { "edit": "ask", "bash": { "git push*": "ask", "*": "allow" } },
  "mcp": {
    "context7": { "type": "remote", "url": "https://mcp.context7.com/mcp" }
  },
  "instructions": ["CONTRIBUTING.md", "docs/guidelines.md"]
}
```

Project rules live in **`AGENTS.md`** (an emerging cross-tool convention, also read by Codex, Cursor, Hermes and others; OpenCode also falls back to `CLAUDE.md`).

### 7.5 Pros / Cons

| ✅ Pros | ❌ Cons |
|---|---|
| Truly provider-agnostic; use subscriptions or any API, or local models (Ollama/LM Studio) | Quality depends on chosen model/prompt tuning per provider |
| Open source, hackable, plugin & custom tool system | Fewer enterprise controls than Claude Code |
| Great TUI, LSP-aware, multi-session, shareable sessions | Rapid release cadence → occasional breaking changes |
| Client/server design enables remote & scripted use | Smaller ecosystem of curated skills/plugins |
| No vendor lock-in | Some provider-subscription auth routes may be restricted by providers' ToS ⚠️ |

**Use cases:** same as Claude Code but when you need model freedom, cost control, privacy (local models), or an open stack; multi-provider benchmarking; self-hosted team coding assistant.

---

## 8. n8n

**What it is:** A **workflow automation platform** (visual node editor, 400+ integrations) with first-class **AI Agent nodes** (LangChain-based), plus MCP client/server nodes. Fair-code licensed, self-hostable (Docker) or cloud. It's the "glue": deterministic orchestration with LLM reasoning where needed.

### 8.1 Architecture

```
 Trigger (Webhook · Cron · Chat · Email · App event)
        │
        ▼
 ┌─────────────── Workflow (DAG of nodes) ───────────────┐
 │ Regular nodes: HTTP · Code (JS/Python) · IF · Switch  │
 │                Merge · Loop · Set · DB · SaaS apps    │
 │                                                        │
 │  ┌──────────── AI Agent node ───────────────┐          │
 │  │ Chat Model   (OpenAI/Anthropic/Ollama…)  │          │
 │  │ Memory       (Window buffer / Postgres / │          │
 │  │               Redis …)                   │          │
 │  │ Tools        (any n8n node, HTTP, Code,  │          │
 │  │               sub-workflow, MCP Client)  │          │
 │  │ Output parser (structured JSON)          │          │
 │  └──────────────────────────────────────────┘          │
 └────────────────────────────────────────────────────────┘
        │
        ▼
 Output: Respond to Webhook · Slack · Email · DB write · …
```

Also: **MCP Server Trigger** node (expose n8n workflows as tools to Claude Code etc.), **MCP Client Tool** node (use external MCP servers), **Execute Workflow** (sub-workflows as tools), **Human-in-the-loop** approval steps ⚠️.

### 8.2 Run / Operate

```bash
# Quick start
npx n8n                                  # local, http://localhost:5678

# Docker
docker volume create n8n_data
docker run -it --rm --name n8n -p 5678:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n

# Production: docker-compose with Postgres + queue mode (Redis + workers)
```

| Env var / setting | Purpose |
|---|---|
| `N8N_ENCRYPTION_KEY` | Encrypts stored credentials: **back it up!** |
| `WEBHOOK_URL` | Public base URL for webhooks |
| `DB_TYPE=postgresdb` + `DB_POSTGRESDB_*` | Production DB (instead of SQLite) |
| `EXECUTIONS_MODE=queue` | Scale with Redis + workers |
| `N8N_BASIC_AUTH_*` / SSO | Access control ⚠️ (varies by version) |
| `EXECUTIONS_DATA_PRUNE=true` | Clean old execution data |

CLI helpers: `n8n export:workflow --all`, `n8n import:workflow --input=…`, `n8n export:credentials` (version your workflows in git).

### 8.3 Agent Node Design Tips

1. **One clear job per agent;** give it a tight system message (role, rules, output format).
2. **Few, well-named tools** with precise descriptions: the description is the tool's prompt.
3. **Wrap risky actions** in sub-workflows with validation and approval.
4. **Use structured output parser** for reliable JSON downstream.
5. **Set max iterations** and timeouts; add Error Trigger workflow for alerts.
6. **Persistent memory** keyed by session/user ID (Postgres/Redis), not default in-memory for production.
7. **Pin test data** and use execution history to debug; test with real edge cases.
8. **Prefer deterministic nodes** for anything that doesn't need judgment (cheaper, faster, auditable).

### 8.4 Common Patterns

| Pattern | Flow |
|---|---|
| **Email triage** | Gmail trigger → AI classify (structured) → Switch → label / draft reply / create ticket |
| **RAG chatbot** | Chat trigger → Agent + Vector Store tool (Qdrant/Pinecone/pgvector) → reply |
| **Lead enrichment** | Webhook → HTTP enrich → Agent scoring → CRM update → Slack alert |
| **Content pipeline** | Cron → RSS fetch → LLM summarize → human approval → publish |
| **Agent-as-a-tool** | Orchestrator agent calls sub-workflows (each an agent/tool) |
| **Coding-agent bridge** | n8n MCP Server Trigger exposes business workflows to Claude Code / OpenCode |
| **Monitoring** | Schedule → fetch metrics → anomaly check (LLM) → PagerDuty/Slack |

### 8.5 Pros / Cons

| ✅ Pros | ❌ Cons |
|---|---|
| Visual, fast to build, huge integration catalog | Not a coding agent; weak at open-ended repo-level tasks |
| Deterministic + AI hybrid; auditable executions | Complex workflows become hard to maintain ("spaghetti canvas") |
| Self-hostable, data stays in-house | Fair-code license (not OSI open source); enterprise features paywalled |
| Webhooks/cron/triggers built in (proactive) | Agent node autonomy/loop control less flexible than code frameworks |
| MCP both directions, sub-workflows as tools | Scaling needs queue mode + ops knowledge; version-control UX is clunky |

**Use cases:** business process automation, support/sales ops, data sync, scheduled reports, chatbots, alerting, glue between systems, exposing safe "tool endpoints" to coding agents.

---

## 9. Hermes Agent (Nous Research)

**What it is:** An open-source (MIT), **self-improving, persistent autonomous agent** from Nous Research, released **February 2026**. Marketed as *"The agent that grows with you."* It lives on your machine or server, talks to you through **messaging gateways** (Telegram, Discord, Slack, WhatsApp, …) and CLI, remembers across restarts, and **turns experience into reusable skills**. Model-agnostic (OpenRouter, Nous Portal, OpenAI/Anthropic-compatible endpoints, local models). Linux, macOS, WSL2 supported. Nous reports very rapid adoption (hundreds of thousands of GitHub stars; ⚠️ figures vary by source) and raised funding (reported ~$90M at a ~$1.5B valuation) to build an enterprise offering while promising to keep the core open source.

### 9.1 Architecture

```
┌──────────── Channels ─────────────┐
│ CLI · Telegram · Discord · Slack  │
│ WhatsApp · voice/wake word · API  │
│ webhooks (signed) · A2A links     │
└───────────────┬───────────────────┘
                ▼
┌──────────────── Hermes core / gateway ──────────────────────┐
│  Agent loop (default tool-iteration cap raised 90 → 500     │
│  in v0.20 ⚠️)                                                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ LEARNING LOOP                                          │ │
│  │  1. Persistent memory (facts, preferences, projects)   │ │
│  │  2. Session search (recall past conversations)         │ │
│  │  3. Skill creation/refinement from completed tasks     │ │
│  └────────────────────────────────────────────────────────┘ │
│  Tools: terminal · files · web · browser · image gen · TTS  │
│  Scheduler (cron-like automations) · Subagents · MCP        │
│  Execution backends: local · Docker · SSH · serverless      │
└───────────────┬─────────────────────────────────────────────┘
                ▼
   Model providers (OpenRouter · Nous Portal · local · others)
   Research hooks: batch trajectory generation · Atropos RL envs
```

### 9.2 Distinguishing Ideas

| Idea | Meaning |
|---|---|
| **Closed learning loop** | After solving something, the agent can write a **skill** (procedure) so next time it's faster and more reliable |
| **Persistent identity & memory** | Not session-bound; remembers user, projects, preferences |
| **Runs anywhere** | $5 VPS, home server, Docker, serverless: agent is decoupled from your laptop |
| **Model portability** | Switch model/provider without losing memory/skills ("own the harness, rent the model") |
| **Research-ready** | Trajectory generation/compression and RL environments (Atropos) to train better tool-calling models |
| **Nous Portal** (optional paid) | Bundles model access + tools (web search, image gen, TTS, hosted browser) |

### 9.3 Typical Setup Flow ⚠️ (verify exact commands in the README)

```bash
# 1. Install via the official one-line installer from the repo README (Linux/macOS/WSL2)
# 2. First-run setup wizard: choose model provider, API key, messaging platform
hermes setup           # (wizard)
hermes                 # start interactive CLI chat
hermes model           # choose/switch provider & model
hermes gateway         # run the messaging gateway (Telegram/Discord/Slack…)
hermes skills          # browse / manage skills
hermes update          # update
hermes doctor          # diagnose
```

Inside chat, slash-style commands manage sessions, models and skills (e.g. `/new`, `/model`, `/skills`, `/compress`, `/retry`, `/undo`). ⚠️ Names evolve; run `/help`.

### 9.4 Best Practices

1. **Start in a sandbox** (Docker/VPS with no secrets) before giving access to real accounts.
2. **Allow-list chat users** on every messaging gateway; never leave a bot open to the public.
3. **Review auto-generated skills** periodically: prune, edit, version them in git (skill drift & poisoning are real risks).
4. **Use the strongest model for hard tasks, cheap model for routine runs**; set budgets/iteration caps (the 500-iteration default can burn tokens).
5. **Give it a narrow first job** (daily briefing, repo watcher), then expand.
6. **Back up memory + skills directories.**
7. **Treat any inbound webhook/message as untrusted input** (prompt injection).

### 9.5 Pros / Cons

| ✅ Pros | ❌ Cons |
|---|---|
| Genuine cross-session learning (memory + self-written skills) | Young project: fast-moving, breaking changes, uneven docs |
| MIT, no telemetry, self-hosted, no vendor lock-in | Security, hardening and maintenance are **your** responsibility |
| Reaches you where you are (chat apps, voice) | Self-improvement quality varies; bad skills can compound |
| Model-agnostic; cheap to run on small servers | Less polished for deep IDE coding than Claude Code/OpenCode |
| Scheduler + subagents + MCP → proactive automation | Long autonomous loops → cost surprises |
| Backed by an active research org | Commercial direction (enterprise/Portal) may shift roadmap |

### 9.6 Use Cases
Personal ops assistant (briefings, reminders, inbox digests) · always-on research agent · server/DevOps companion over SSH · automated monitoring with chat alerts · knowledge accumulation (notes → skills) · voice-driven assistant · training-data generation for tool-use models.

---

## 10. OpenClaw

> The user wrote "OpenCloaw": this refers to **OpenClaw** (formerly **Clawdbot → Moltbot/Molty → OpenClaw**), the viral self-hosted personal AI assistant. Hermes Agent is often described as its main open-source rival.

**What it is:** A long-lived **Node.js "Gateway" daemon** on your own hardware that connects messaging channels (WhatsApp, Telegram, Discord, Slack, Signal, iMessage-style bridges, …) to an **embedded agent runtime** (built on the Pi agent core ⚠️). It has file-based persona/memory, a skills system, cron scheduling and heartbeats. Philosophy: **behavior = editable Markdown files**: no code changes or redeploys.

### 10.1 Architecture

```
 You ── WhatsApp / Telegram / Discord / Slack / Signal …
                   │
                   ▼
 ╔═══════════════ GATEWAY (single daemon, default port 18789) ═══════════════╗
 ║ Channel adapters · message queue · auth/allow-lists · REST/dashboard      ║
 ║ Cron scheduler · session store (SQLite) · channel workers                 ║
 ╠═══════════════════════════════════════════════════════════════════════════╣
 ║ Embedded agent runtime  (1 agent per gateway, 1 workspace per agent)      ║
 ║   System prompt = base + SOUL.md + per-run overrides                      ║
 ║   Bootstrap files: AGENTS.md · SOUL.md · TOOLS.md · IDENTITY.md ·         ║
 ║                    USER.md · BOOTSTRAP.md (one-shot, self-deleting)       ║
 ║   Memory: MEMORY.md (curated) + daily notes + SQLite transcripts          ║
 ║   LLM router → Anthropic / OpenAI / Google / local …                      ║
 ╠═══════════════════════════════════════════════════════════════════════════╣
 ║ Skills (SKILL.md folders; 6-level precedence: workspace → project agent   ║
 ║ → personal agent → managed ~/.openclaw/skills → bundled → extra dirs)     ║
 ║ Tools: shell/exec (with approvals) · browser · files · web · sub-agents   ║
 ║ Sandbox + tool policy + exec approvals                                    ║
 ╚═══════════════════════════════════════════════════════════════════════════╝
 State lives in ~/.openclaw/  → backing up that dir = backing up OpenClaw
```

### 10.2 Workspace Files

| File | Purpose |
|---|---|
| `SOUL.md` | Identity, personality, tone, **decision boundaries** (when to act vs. escalate) |
| `AGENTS.md` | Operating instructions, safety rules, routing/team rules |
| `USER.md` | Facts about *you* (preferences, timezone, context) |
| `TOOLS.md` | Environment specifics: SSH hosts, devices, API endpoints (keep *out* of skills for portability) |
| `IDENTITY.md` | Name/avatar/handle details |
| `MEMORY.md` | Curated long-term memory |
| `memory/YYYY-MM-DD.md` | Daily logs |
| `HEARTBEAT.md` | Standing checklist the agent runs on periodic heartbeats |
| `BOOTSTRAP.md` | First-run onboarding script; agent deletes it afterwards |

### 10.3 Commands (⚠️ verify with `openclaw --help`; early docs used `clawdbot …`)

```bash
openclaw onboard                # guided first-time setup (workspace, model, channels)
openclaw gateway start|stop|restart|status   # manage the daemon; restart = reload config
openclaw dashboard              # open the inspection UI
openclaw skills list            # installed skills
openclaw sessions list          # active sessions
openclaw agent spawn <name> --session isolated --model <model>   # sub-agent (per community template)
openclaw doctor                 # diagnose config
# Config: ~/.openclaw/openclaw.json ; cron jobs: ~/.openclaw/cron/jobs.json
```

### 10.4 Context Budget (default allocation in one documented architecture)
System prompt 15% · recent conversation 40% · long-term memory 20% · tool definitions 15% · reserved for response 10%.

### 10.5 Best Practices

1. **Never expose the Gateway/dashboard publicly.** Bind to localhost, use a VPN/Tailscale/SSH tunnel, require auth.
2. **Strict channel allow-lists** (only your own accounts/numbers).
3. **Run in a VM/container/dedicated machine** with minimal credentials (a spare Mac mini/VPS is the common pattern).
4. **Use exec approvals + tool policy**: default to "ask" for shell, network, file-delete, money, outbound messages.
5. **Vet every third-party skill/plugin** (read source; marketplaces have hosted malicious skills): prefer writing your own.
6. **Encode escalation rules in `SOUL.md`** ("never send email without confirmation", "spend limit €X").
7. **Curate `MEMORY.md`**: periodic review so it doesn't rot or accumulate secrets.
8. **Separate secrets from workspace files**; don't put tokens in Markdown that gets sent to LLM providers.
9. **Back up `~/.openclaw/`** (git for the workspace; encrypted backup for state).
10. **Keep heartbeat checklists short** (every heartbeat costs tokens).

### 10.6 Pros / Cons

| ✅ Pros | ❌ Cons |
|---|---|
| Lives in the chat apps you already use; always-on, proactive | **Largest security surface** of all tools here: broad system access + untrusted inbound messages |
| Behavior fully editable as text; no redeploy | Reports of misconfigured/exposed instances and malicious community skills ⚠️ |
| Persistent file-based memory; multi-agent routing | Token cost of heartbeats/cron/long contexts adds up |
| Large community, many skills, multi-provider | Rapid renames/changes (Clawdbot→Moltbot→OpenClaw) → fragmented docs |
| Self-hosted, you own the data | Reliability of autonomy varies; needs babysitting early on |

### 10.7 Use Cases
Personal "chief of staff" (calendar, email triage, reminders) · home/server automation by chat · multi-agent content pipeline (research → draft → edit → publish) · always-on monitoring & alerts · voice/phone-style assistant · small-team ops bot.

---

## 11. Other Tools Worth Knowing

| Tool | One-liner | Notes |
|---|---|---|
| **OpenAI Codex CLI** | OpenAI's open-source terminal coding agent | Reads `AGENTS.md`; sandboxed execution; approval modes |
| **Gemini CLI** | Google's open-source terminal agent | Large context, `GEMINI.md`, MCP support, free tier ⚠️ |
| **Aider** | Git-native pair programmer in terminal | Auto-commits, repo map, great with many models; less "agentic" |
| **Cline / Roo Code / Kilo Code** | VS Code agent extensions | Human-approval step-by-step; BYO model |
| **Cursor / Windsurf / GitHub Copilot agent mode** | AI-first IDEs / agent modes | Background agents, PR creation |
| **Goose (Block)** | Open-source on-machine agent | MCP-centric extensions |
| **Amp, Factory Droids, Devin, Jules** | Hosted/autonomous SWE agents | Run in cloud sandboxes, produce PRs |
| **Claude Agent SDK** | Claude Code's engine as a library | Python/TS; hooks, subagents, MCP |
| **OpenAI Agents SDK** | Lightweight multi-agent framework | Handoffs, guardrails, tracing |
| **LangGraph (LangChain)** | Stateful graph-based agent orchestration | Durable execution, HITL checkpoints, production favorite |
| **CrewAI** | Role-based multi-agent crews | Fast to prototype; opinionated |
| **AutoGen / Microsoft Agent Framework** | Conversational multi-agent | Enterprise integration |
| **LlamaIndex / Haystack** | RAG-first frameworks | Data connectors, retrieval pipelines |
| **Dify / Flowise / Langflow** | Low-code LLM app builders | Visual alternatives to n8n for AI-centric apps |
| **Zapier / Make** | SaaS automation | Hosted n8n alternatives |
| **A2A (Agent2Agent)** | Protocol for agents to talk to each other | Complements MCP (tools) with agent↔agent interop |
| **AGENTS.md** | Cross-tool instruction file convention | Use one file; symlink/import for tool-specific names |

---

## 12. Comparison Matrix

| Dimension | Claude Code | OpenCode | n8n | Hermes Agent | OpenClaw |
|---|---|---|---|---|---|
| **Core job** | Software engineering | Software engineering | Workflow automation | Persistent self-improving assistant | Always-on personal assistant |
| **Interface** | Terminal/IDE/web | TUI/desktop/IDE | Visual canvas | CLI + chat apps | Chat apps + dashboard |
| **Autonomy style** | Supervised → autonomous (headless) | Supervised → autonomous | Deterministic + agent nodes | Autonomous, proactive | Autonomous, proactive |
| **Proactive (cron/heartbeat)** | Via CI/headless/schedules | Via scripting | ✅ native | ✅ native | ✅ native |
| **Memory** | `CLAUDE.md` + auto-memory | `AGENTS.md` + sessions | Memory nodes (buffer/DB) | Persistent memory + session search + skills | Markdown files + SQLite |
| **Skills support** | ✅ native | Partial/compatible ⚠️ | ✗ (use sub-workflows) | ✅ + self-authoring | ✅ + 6-level precedence |
| **MCP** | Client | Client | Client + Server | Client | Client/plugins ⚠️ |
| **Models** | Claude | Any | Any | Any | Any |
| **License** | Proprietary | Open source | Fair-code | MIT | Open source |
| **Self-host data** | Local files; API calls out | ✅ (with local LLM: fully) | ✅ | ✅ | ✅ |
| **Setup effort** | Low | Low | Medium | Medium | Medium–High |
| **Security risk (relative)** | Low–Med (strong permission model) | Low–Med | Low–Med | Med–High | **High** if misconfigured |
| **Best at** | Code quality, safety controls | Flexibility, openness | Integrations, reliability | Learning over time | Reach & proactivity |
| **Biggest risk** | Cost / vendor | Model-dependent quality | Workflow sprawl | Skill drift, maturity | Exposure, prompt injection |

---

## 13. Decision Guide: Which Tool When?

```
Need to change code in a repo?
 ├─ Want best-in-class quality + safety, fine with Claude models → Claude Code
 ├─ Want any model / local / open source / cost control          → OpenCode (or Codex CLI / Gemini CLI / Aider)
 └─ Want tiny human-approved steps inside IDE                    → Cline / Roo / Copilot agent mode

Need to automate business processes across SaaS apps?
 ├─ Steps known, audit + reliability needed → n8n (Zapier/Make if hosted-only)
 └─ Some fuzzy judgement in the middle      → n8n AI Agent node (+ approval step)

Need an assistant that is ALWAYS on, reachable via chat, remembers you?
 ├─ Want it to learn/refine its own skills, MIT, model-portable → Hermes Agent
 ├─ Want the largest community/skill ecosystem, accept more hardening work → OpenClaw
 └─ Corporate/regulated → prefer managed/enterprise agents + strict sandboxing

Need reusable expertise across tools?        → Agent Skills (+ AGENTS.md)
Need to connect to external systems/data?    → MCP servers (or n8n as MCP server)
Need guaranteed behavior (format, block rm)? → Hooks / permission rules / sandbox, NOT prompts
Building your own product agent?             → Claude Agent SDK / OpenAI Agents SDK / LangGraph
```

**Popular combos:**
- **Claude Code + n8n (MCP):** coding agent triggers vetted business workflows.
- **Claude Code + Skills + Hooks:** team-standard, enforceable engineering workflow.
- **n8n (cron/webhook) → Claude Code headless:** scheduled repo maintenance (dependency bumps, doc refresh).
- **Hermes/OpenClaw + MCP + Skills:** personal always-on assistant with controlled tool access.
- **OpenCode + local model (Ollama):** private/offline dev assistant.

---

## 14. Best Practices (Universal)

### 14.1 The 12 Golden Rules

1. **Start simple.** Single agent + good tools beats a clever multi-agent swarm. Add complexity only when measured failures demand it.
2. **Verification is the force multiplier.** Give the agent a way to check its own work: tests, linters, type-checkers, screenshots, schema validators, a rubric. *Unverifiable tasks produce confident garbage.*
3. **Explore → plan → act.** Ask for research and a plan first (plan mode); approve; then execute.
4. **Manage context like a budget.** Fresh sessions per task; compact/clear often; offload to files/subagents; avoid dumping huge logs.
5. **Write durable instructions once** (`CLAUDE.md`/`AGENTS.md`/`SOUL.md`); keep them short, concrete, tested.
6. **Use the right extension for the job:** rules → instruction file · procedures → skill · live systems → MCP · guarantees → hook/permission · isolation → subagent.
7. **Least privilege.** Narrow tool allow-lists, read-only by default, scoped tokens, no prod credentials.
8. **Sandbox autonomy.** The more autonomous, the more isolated (container/VM, no secrets, network egress limits).
9. **Checkpoint & make reversible.** Git commits/branches/worktrees; backups; rewind/undo.
10. **Cap loops.** Max turns/iterations, timeouts, budget limits, loop-detection.
11. **Log everything.** Transcripts, tool calls, costs: you can't improve or audit what you can't see.
12. **Keep a human at the irreversible edges.** Payments, deletions, prod deploys, external messages, legal/medical actions.

### 14.2 Task Design for Agents

| Do | Don't |
|---|---|
| Define **goal + constraints + definition of done** | "Make it better" |
| Point to **specific files/examples/patterns** | Hope it finds conventions by guessing |
| Give **commands** to run tests/build | Assume it knows your toolchain |
| Break big work into **independently verifiable chunks** | One 3-hour mega-prompt |
| State **what NOT to touch** | Leave blast radius implicit |
| Ask it to **ask clarifying questions** when ambiguous | Force guesses on critical decisions |
| Provide **screenshots / error logs / failing test** | Paraphrase the error from memory |

### 14.3 Prompt Template for Autonomous Tasks

```text
GOAL: <one-sentence outcome>
CONTEXT: <where to look: files, docs, tickets, @refs>
CONSTRAINTS: <style, libs, don't-touch list, performance/security rules>
STEPS (suggested): 1) investigate 2) propose plan 3) implement 4) verify
VERIFY: run `<test/lint/build cmd>`; all must pass. Also <manual check>.
DONE WHEN: <acceptance criteria>
IF BLOCKED: stop and ask; do not guess about <risky area>.
OUTPUT: short summary of changes, risks, and follow-ups.
```

### 14.4 Team & Org Practices

- **Commit `.claude/` / `AGENTS.md` / skills** to the repo; review them like code.
- Create a **shared skills/plugin library** with owners and versioning.
- **Central policy** (enterprise settings, managed MCP allow-lists).
- Keep **an "agent changelog"**: what it changed and why, in PR descriptions.
- Pair **junior devs + agents** with mandatory human review.
- Track **metrics:** task success rate, review rework rate, tokens per merged PR, time-to-merge.

---

## 15. Prompting & Context Engineering

| Technique | Use | Example |
|---|---|---|
| **Role + rules in system/instruction file** | Stable behavior | "You are a careful reviewer; never skip tests" |
| **Few-shot examples** | Format/style adherence | Show 2 good commits / reports |
| **Think-before-act** | Complex reasoning | "think hard / ultrathink" style keywords or extended thinking ⚠️ model/tool-specific |
| **Plan mode** | Prevent premature edits | Read-only until approved |
| **Structured output** | Machine-consumable results | JSON schema / output parser |
| **XML/Markdown sectioning** | Disambiguate parts of prompt | `<context>…</context><task>…</task>` |
| **Negative constraints + reasons** | Avoid pitfalls | "Don't mock the DB: integration tests caught real bugs before" |
| **Progressive disclosure** | Save context | Skills, linked reference files |
| **Context compaction** | Long sessions | `/compact focus on the auth refactor` |
| **Scratchpad files** | Long-horizon tasks | `PLAN.md`, `PROGRESS.md`, `TODO.md` updated by the agent |
| **Subagent offloading** | Keep main context clean | "Use a subagent to find all usages of X and summarize" |
| **Checklists** | Reliability | Copy-paste checklist the agent ticks off |

### Long-Horizon Task Pattern (hours-long autonomy)

```
1. Initializer step: write SPEC.md, PLAN.md (milestones), tests/feature list (all initially failing)
2. Loop:  pick next unfinished item → implement → run verification → update PROGRESS.md → commit
3. Each fresh context starts by reading SPEC.md + PROGRESS.md + `git log`
4. Stop condition: all items pass OR max iterations OR budget hit
5. Final: summary + open questions for human
```

---

## 16. Security & Safety Checklist

### 16.1 Threat Model (what actually goes wrong)

| Threat | Description | Mitigation |
|---|---|---|
| **Prompt injection (direct/indirect)** | Malicious text in web pages, emails, issues, files, tool outputs hijacks the agent | Treat all external content as untrusted; restrict tools after reading untrusted data; human approval for sensitive actions; separate "reader" and "actor" agents |
| **Lethal trifecta** | Access to private data + exposure to untrusted content + ability to exfiltrate/communicate externally = exploitable | **Remove at least one leg** (e.g. no outbound network, or no private data) |
| **Malicious skills/plugins/MCP servers** | Supply-chain attacks; tool poisoning; hidden instructions in descriptions | Audit source, pin versions, allow-list, run in sandbox, prefer first-party |
| **Credential leakage** | Secrets in prompts/memory/logs/`.env` | Deny-read on secret paths, secret managers, short-lived scoped tokens, log redaction |
| **Destructive actions** | `rm -rf`, `DROP TABLE`, force-push, wrong-env deploy | Permission deny rules, PreToolUse hooks, read-only DB roles, branch protection, backups |
| **Exposed agent endpoints** | Public gateways/dashboards/webhooks w/o auth | Bind to localhost, VPN, auth, allow-lists, signed webhooks |
| **Runaway cost/loops** | Infinite tool loops, huge contexts | Max turns, budgets, alerts, loop detection |
| **Memory poisoning** | Bad facts/skills persist and compound | Review memory/skills, provenance, periodic cleanup, versioning in git |
| **Over-trust / automation bias** | Humans rubber-stamp agent output | Mandatory review for risky changes, tests as gate, diff-size limits |
| **Data residency / privacy** | Sensitive data sent to third-party model APIs | Local models, zero-retention agreements, redaction, policy by data class |

### 16.2 Hardening Checklist

- [ ] Run agents in **containers/VMs**, non-root, no host mounts beyond the project
- [ ] **Network egress allow-list** (package registries, your APIs only)
- [ ] **No production credentials**; use staging, read-only, scoped, expiring tokens
- [ ] **Deny rules** for `.env`, `~/.ssh`, cloud credentials, browser profiles
- [ ] **PreToolUse hooks** block dangerous commands & secret exfil patterns
- [ ] `--dangerously-skip-permissions` / "YOLO" modes **only** in throwaway sandboxes
- [ ] **Allow-list** inbound chat identities; require pairing/approval for new users
- [ ] **Sign & verify webhooks**; rate-limit; validate payloads
- [ ] **Pin & review** MCP servers, skills, plugins; no `curl | bash` from unknown sources
- [ ] **Audit logs** retained; alert on anomalies (unusual tool use, big spend)
- [ ] **Backups + rollbacks** for data the agent can touch
- [ ] **Separate identities/accounts** for the agent (its own email, GitHub user, API keys)
- [ ] Periodic **red-team**: feed it injected docs/issues and see what it does

---

## 17. Typical Use Cases & Recipes

### 17.1 Use-Case Map

| Domain | Use case | Best-fit tool(s) |
|---|---|---|
| **Software** | Feature dev, bugfix, refactor, migration | Claude Code, OpenCode, Codex CLI |
| | PR review bot, test generation, docs | Claude Code headless / GitHub Action, n8n + agent |
| | Legacy code comprehension | Claude Code, OpenCode (plan mode) |
| **DevOps** | Incident triage, log analysis, IaC edits | Claude Code + MCP (Sentry/Datadog), Hermes over SSH |
| **Business ops** | Email/ticket triage, CRM enrichment, reporting | n8n (+ agent node) |
| **Research** | Literature/market scans, competitor tracking | Hermes, OpenClaw, Claude Code + search MCP, deep-research modes |
| **Personal productivity** | Briefings, reminders, calendar, notes | Hermes, OpenClaw |
| **Content** | Research → draft → edit → publish pipeline | n8n pipeline or OpenClaw multi-agent |
| **Data** | Cleaning, analysis, dashboards, one-off scripts | Claude Code, Skills (xlsx/pdf/docx) |
| **Customer support** | RAG chatbot, triage & escalation | n8n + vector store, LangGraph |
| **Knowledge mgmt** | Turn notes into structured wiki | Claude Code/Hermes + Markdown vault |
| **Security** | Code audit, dependency review, triage (with care) | Claude Code (read-only), dedicated sandboxed agents |
| **Education/training** | Tutor agents, exercise generation | Skills + chat agents |

### 17.2 Recipe: Safe Autonomous Coding Setup (Claude Code)

```text
1. Repo: write CLAUDE.md (commands, conventions) + .claude/settings.json (allow/deny)
2. Add hooks: PostToolUse → formatter; PreToolUse → block rm -rf / secrets reads
3. Add skills: release-process, db-migration-checklist
4. Add subagents: code-reviewer, test-writer
5. Work in a git worktree; run `claude` → Shift+Tab (plan mode) → approve plan
6. Let it implement; it runs tests (verification loop)
7. Fresh session: "review the diff against main" (writer/reviewer pattern)
8. Human reviews, merges
```

### 17.3 Recipe: Scheduled Repo Maintenance (n8n → Claude Code headless)

```text
n8n Cron (weekly) → SSH/Execute node on a sandbox runner:
   claude -p "Update minor deps, run tests, and if green open a PR titled 'chore: deps'"
          --allowedTools "Bash(npm:*)" "Bash(git:*)" "Bash(gh pr create:*)" "Edit" "Read"
          --max-turns 30 --output-format json
→ n8n parses JSON → Slack summary (success/failure + PR link) → Error Trigger on failure
```

### 17.4 Recipe: Email Triage Agent (n8n)

```text
Gmail Trigger → AI Agent (system: classify {urgent, billing, spam, lead, other}; JSON output)
→ Switch on category
   urgent  → Slack DM + create task
   billing → forward to finance + auto-ack draft (human approves)
   lead    → CRM create + enrichment sub-workflow
   spam    → label & archive
→ Log to Postgres (for evals). Never auto-send without approval at first.
```

### 17.5 Recipe: Personal Daily Briefing (Hermes / OpenClaw)

```text
Cron 07:30 → agent gathers: calendar (MCP), unread important mail (filtered), weather,
RSS/news topics from USER.md, open tasks → writes a 10-line brief → sends to Telegram.
Rules (SOUL/skill): never reply to emails; never share contents externally; ask before any write action.
```

### 17.6 Recipe: Build a Skill from a Repeated Task

```text
1. Do the task once with the agent, correcting it until output is right
2. Ask: "Summarize what worked as a SKILL.md with a precise description; move fixed logic into scripts/"
3. Add a verification step + a short example
4. Test in a FRESH session on 3 varied inputs; refine description if it doesn't trigger
5. Commit to repo / share via plugin marketplace
```

### 17.7 Recipe: Expose Business Actions to a Coding Agent Safely (n8n MCP)

```text
n8n workflow "create-jira-ticket" (validates inputs, fixed project, rate-limited)
→ MCP Server Trigger node exposes it as a tool
→ claude mcp add n8n --transport http <n8n-mcp-url>   (with auth header)
→ Agent can create tickets but cannot do arbitrary Jira writes.
```

### 17.8 Recipe: Parallel Agents with Git Worktrees

```bash
git worktree add ../app-auth   -b feat/auth
git worktree add ../app-billing -b feat/billing
(cd ../app-auth    && claude "Implement OAuth login per docs/auth.md")   # terminal 1
(cd ../app-billing && claude "Add invoice PDF export per docs/billing.md") # terminal 2
# Merge each via PR; clean up: git worktree remove ../app-auth
```

---

## 18. Failure Modes & Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Agent "forgets" rules mid-session | Context overflow / compaction loss | Shorten instruction file; `/clear` per task; put must-haves in **hooks**; restate key constraints |
| Ignores a skill | Weak/vague `description` | Rewrite with concrete trigger phrases; keep name unique; test in clean session |
| Skill triggers wrongly | Description too broad | Narrow scope; split skill |
| Loops / repeats same failing action | No new info, unclear success criterion | Cap turns; add verification; tell it to stop & summarize blockers; try different model/approach |
| Edits wrong files / over-scopes | Vague task, no boundaries | Specify files & "don't touch"; plan mode first; smaller tasks |
| Tests pass but behavior wrong | Weak/mocked tests; agent edited tests | Forbid modifying tests; add integration tests; review diffs |
| Hallucinated APIs/packages | Missing docs context | Docs MCP (e.g. Context7), paste docs, require compile/test; verify packages exist (slopsquatting risk) |
| Costs spike | Long loops, huge context, expensive model everywhere | Budgets, cheaper model for subtasks, subagents, `/compact`, cache-friendly prompts |
| MCP tools not showing | Wrong transport/scope/auth | `/mcp` or `mcp list`; check logs; re-auth; restart session |
| Hooks not firing | Matcher/event mismatch, bad exit codes | Print debug; validate JSON stdin handling; exit 2 to block |
| Permission prompts constantly | Rules too tight/unspecified | Add safe `allow` rules for repeated commands (e.g. test/lint) |
| n8n agent returns malformed JSON | No output parser / loose prompt | Add structured output parser + retry/repair node |
| n8n webhook unreachable | `WEBHOOK_URL`/proxy misconfig | Set correct public URL, check reverse proxy |
| Gateway agent unresponsive (OpenClaw/Hermes) | Daemon down, channel token expired, model quota | Check status/logs, restart gateway, verify credentials & provider limits |
| Memory bloat / stale facts | No curation | Periodic review; summarize; delete outdated; version in git |
| Different results between runs | Non-determinism | Lower temperature where possible; deterministic scripts; evals over many runs |

---

## 19. Cost & Performance Tuning

| Lever | How |
|---|---|
| **Right-size the model** | Strong model for planning/hard reasoning; small/fast for search, formatting, summaries (subagents with `model:` override; `small_model` in OpenCode) |
| **Prompt caching** | Keep stable prefix (system prompt, tool defs, instruction file) unchanged to hit caches; avoid reshuffling tool lists |
| **Shrink context** | Short instruction files, skills instead of giant prompts, fewer MCP servers, targeted greps vs. whole-file reads |
| **Fresh sessions** | `/clear` between unrelated tasks |
| **Constrain loops** | `--max-turns`, iteration caps, budgets, timeouts |
| **Batch & parallelize wisely** | Parallel subagents save wall-clock, **not** tokens |
| **Use deterministic code** | Scripts/regex/SQL where possible; LLM only for judgment |
| **Local models** | For high-volume, low-stakes steps (classification, extraction) via Ollama/LM Studio |
| **Cache tool results** | Don't re-fetch same docs; store in files |
| **Monitor** | `/cost`, provider dashboards, n8n execution stats, per-task token metrics |

**Rule of thumb:** Cost ≈ (context size) × (number of turns) × (model price). Cut any factor.

---

## 20. Evaluation & Observability

### 20.1 What to Measure

| Metric | Why |
|---|---|
| **Task success rate** (verified by tests/rubric) | Core quality |
| **Pass@k / consistency** | Reliability across runs |
| **Tokens & $ per task** | Efficiency |
| **Steps/tool calls per task** | Detect flailing |
| **Human rework rate** (review comments, reverts) | Real-world value |
| **Time to completion** | Productivity |
| **Safety incidents / blocked actions** | Guardrail effectiveness |
| **Skill/tool trigger accuracy** | Are the right skills/tools chosen? |

### 20.2 Practical Eval Loop

1. Collect **10-30 real tasks** with known-good outcomes (a mini benchmark).
2. Run agent config A vs. B (different CLAUDE.md, skills, model).
3. Score automatically (tests) + LLM-as-judge with rubric + spot-check by humans.
4. Change **one variable** at a time; keep a changelog.
5. Re-run after every model/tool upgrade (regressions happen).

### 20.3 Observability Stack Options
Built-in transcripts & logs (Claude Code, OpenCode, OpenClaw SQLite) · OpenTelemetry export (Claude Code ⚠️ monitoring settings) · LangSmith / Langfuse / Arize Phoenix / Helicone · n8n execution history · provider usage dashboards · custom hooks writing JSONL logs.

---

## 21. Glossary

| Term | Meaning |
|---|---|
| **A2A** | Agent2Agent protocol: standard for inter-agent communication |
| **AGENTS.md** | Cross-tool, plain-Markdown project instructions for agents |
| **Agentic loop** | Observe-think-act cycle repeated until done |
| **Compaction** | Summarizing history to free context |
| **Context engineering** | Deliberately curating what's in the model's context |
| **Function calling / tool use** | Model emits structured calls to developer-defined functions |
| **Gateway** | Daemon bridging chat channels to the agent runtime (OpenClaw, Hermes) |
| **Guardrails** | Checks that constrain inputs/outputs/actions |
| **Harness** | Non-model software that makes an LLM an agent |
| **Heartbeat** | Periodic wake-up that lets an agent act proactively |
| **HITL** | Human-in-the-loop |
| **Lethal trifecta** | Private data + untrusted content + external communication |
| **MCP** | Model Context Protocol |
| **Plan mode** | Read-only mode that produces a plan before edits |
| **Prompt injection** | Malicious instructions embedded in data the agent reads |
| **Progressive disclosure** | Loading info in stages (metadata → body → resources) |
| **RAG** | Retrieval-augmented generation |
| **SOUL.md** | OpenClaw persona/behavior-boundary file |
| **Subagent** | Child agent with isolated context |
| **Trajectory** | Full record of an agent run (thoughts, tool calls, results) |
| **Worktree** | Extra git checkout enabling parallel branches/agents |

---

## 22. Quick-Reference One-Pagers

### 22.1 Claude Code: Top 15 Things to Remember

```
claude                 start       | claude -c / -r      continue / resume
claude -p "…"          headless    | --output-format json   scripting
Shift+Tab              cycle modes (default → accept-edits → plan)
Esc / Esc Esc          interrupt / rewind
/init                  create CLAUDE.md          /memory   edit memory
/clear                 new task, fresh context   /compact  summarize history
/context  /cost        see budget                /model    switch model
/permissions           allow/ask/deny            /mcp      servers & auth
/agents                subagents                 /hooks    guarantees
@file                  reference file            !cmd      run shell inline
.claude/settings.json  team config               .claude/commands/  custom slash cmds
.claude/skills/        skills                    .claude/agents/    subagents
Golden loop: explore → plan → implement → verify (tests) → commit
Never: --dangerously-skip-permissions outside a sandbox
```

### 22.2 SKILL.md Minimal Template

```markdown
---
name: <lowercase-hyphen-name>
description: <What it does>. Use when <specific triggers/keywords>.
---
# <Title>
## When to use
## Steps
1. …
## Verification
- [ ] …
## References
See [details](reference/details.md)
```

### 22.3 n8n Agent Node Checklist

```
[ ] Trigger chosen (webhook / cron / chat / app)      [ ] System message: role + rules + format
[ ] Chat model + temperature set                       [ ] Memory keyed by session/user (DB-backed)
[ ] Tools minimal, well-described                      [ ] Output parser (structured JSON)
[ ] Max iterations + timeout                           [ ] Error Trigger workflow → alert
[ ] Human approval before irreversible actions         [ ] Credentials via n8n creds; ENCRYPTION_KEY backed up
[ ] Workflows exported to git                          [ ] Tested on edge cases with pinned data
```

### 22.4 Always-On Agent (Hermes / OpenClaw) Launch Checklist

```
[ ] Dedicated VM/VPS/container, non-root, minimal secrets
[ ] Gateway bound to localhost / VPN; auth enabled; firewall closed
[ ] Channel allow-list = only you (pairing for others)
[ ] Model provider key with spend cap
[ ] Persona/boundaries written (SOUL.md / skills): what it may do alone vs. must ask
[ ] Exec/tool approvals ON for shell, network writes, outbound messages, money
[ ] Skills: only self-written or audited
[ ] Heartbeat/cron: small, explicit, measured
[ ] Memory + skills dir under git/backup
[ ] Logs reviewed weekly; kill-switch known (stop gateway)
```

### 22.5 Decision One-Liners

| If you want… | Use… |
|---|---|
| Best coding agent, enforceable safety | **Claude Code** |
| Open, model-agnostic coding TUI | **OpenCode** |
| Reliable multi-app automation | **n8n** |
| Reusable expertise packages | **Agent Skills** |
| Standard tool/data plumbing | **MCP** |
| Always-on assistant that learns | **Hermes Agent** |
| Always-on assistant with big community | **OpenClaw** (harden it!) |
| Custom production agent in code | **Claude Agent SDK / LangGraph / OpenAI Agents SDK** |

---

### Final Notes

- **Verify before trusting:** version-specific details (flags, defaults, commands, star counts, valuations) change fast. Use `--help`, `/help`, and official docs.
- **Autonomy should be earned:** start supervised, add verification and logging, then loosen permissions as trust grows.
- **The durable assets** are *your* instruction files, skills, tests, and workflows: the model and harness are swappable.

*End of reference card.*
