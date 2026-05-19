---
name: conjure
description: Use when you want to build a new Claude Code skill, agent, hook, or MCP server but aren't sure which — or when you want to describe what you want in plain language. Auto-detects the right artifact type, asks targeted questions, explains the reasoning, then builds it.
argument-hint: "<describe what you want in plain language>"
allowed-tools: Read,Write,Edit,Bash,Glob,WebSearch,WebFetch
user-invocable: true
---

## Phase 0: Help Check

If `$ARGUMENTS` is `"help"` or `"--help"` or `"-h"`: tell the user to run `/conjure-help` for the full reference, then stop.

---

## Phase 0.5: Load User Preferences

Read both conjure config files if they exist:

```bash
cat ~/.claude/conjure-config.md 2>/dev/null
cat .claude/conjure-config.md 2>/dev/null
```

Follow all instructions found in `## global` and any type-specific sections throughout this command. Project-local instructions (`.claude/conjure-config.md`) take precedence over global ones when they conflict. If neither file exists, proceed with defaults.

---

## Phase 1: Classify + Initial Intake

The user wants to build something. They may or may not know what type.

Their request: **$ARGUMENTS**

Run the classification + intent batch. Use `AskUserQuestion` with these questions (skip any that `$ARGUMENTS` already answers clearly):

**Q1 "Access"** — "Does this need to connect Claude to an external system it can't reach natively (a database, REST API, third-party service like Jira/Linear/Notion)?"
Options:
- Yes — Claude needs live access to something external
- No — it works with what Claude already has access to
- Not sure

**Q2 "Trigger"** — "How does this get activated?" *(skip if Q1 = Yes)*
Options:
- I type a `/command` in the prompt
- Automatically when something happens (file edit, tool call, session start, stop)
- Claude spawns it programmatically with its own context and tools
- Not sure — I just know what I want it to do

**Q3 "Job"** — "What is the ONE thing this does? Be specific — not 'helps with X' but 'does Y when Z happens.'"
Options:
- Connects Claude to an external system / API / database
- Runs a multi-step workflow I trigger manually
- Fires automatically and blocks, logs, or injects context
- Executes a focused research or analysis task in isolation

**Q4 "Scope"** — "Should this work in all projects or just this one?"
Options:
- All projects (user-global)
- This project only
- Not sure

---

## Phase 2: Classify

Based on the answers, determine the artifact type using this table:

| Signal | Type |
|---|---|
| Needs live access to an external system (DB, API, Jira, Notion, Slack, etc.) | **mcp** — MCP server |
| "Connect to", "integrate with", "query", "stop copy-pasting from [service]" | **mcp** — MCP server |
| User types `/command` to trigger it | **skill** |
| Fires on an event (PreToolUse, PostToolUse, SessionStart, Stop, file edit, etc.) | **hook** |
| Claude spawns it with isolated context, a specific model, or restricted tools | **agent** |
| Long-running autonomous research or orchestration Claude initiates | **agent** |
| Unsure but describes a workflow they'd invoke manually | **skill** (default) |

**MCP is not a skill or hook.** A skill can't make authenticated calls to external systems. A hook can't maintain a typed connection to a database. If external access is the core need, it's always MCP.

**MCP + skill together:** If the user needs both a live connection AND a workflow over it, build MCP first, then a skill that calls it.

State the classification and rationale on one line before moving on:

```
→ [type] — because [one sentence reason]
```

---

## Phase 3: Deep-Dive Questions

Based on the classified type, ask a targeted second batch. Use `AskUserQuestion` with up to 4 questions in a single call.

### If type = `agent`

1. Is this a **worker** (one focused task, no subagents) or an **orchestrator** (spawns other agents to parallelize work)?
2. What files or systems does it read? (e.g., specific directories, git history, crash logs, external APIs, internal dashboards)
3. What tools does it need? Check all that apply: Read / Grep / Edit / Bash / Write / WebFetch / Agent (orchestrators only)
4. Does it need to be isolated in a worktree, or can it run against the live working tree? Project-scoped (`.claude/agents/`) or global (`~/.claude/agents/`)?

Additional context to extract if not covered:
- Model: haiku (fast/cheap), sonnet (default), opus (reasoning-heavy)
- Whether it auto-triggers based on a pattern or explicit-only
- Output format: conversation text, file written, both

### If type = `skill`

1. What exact phrase does the user type to trigger it? (This becomes the slash command name and its CSO description.)
2. Does it have side effects — does it commit, push, deploy, delete, or mutate anything?
3. What does it accept as `$ARGUMENTS`? (optional name, a description, flags, nothing?)
4. Does it need to read conversation history or prior context, or can it run in a fresh fork?

Additional context to extract if not covered:
- Skill type: fast (single atomic action), standard (multi-phase workflow), reference (lookup/guidance)
- Tools needed
- Related skills it might be confused with (important for CSO description precision)

### If type = `hook`

1. Which event fires it? (SessionStart / PreToolUse / PostToolUse / UserPromptSubmit / Stop / SubagentStop)
2. Should it **block** execution (exit 2 to cancel the tool call) or just observe/log/notify?
3. What exactly does it check or do — what condition triggers action vs passes silently?
4. Is this always-on or should it be toggleable (via an env var or flag file)?

Additional context to extract if not covered:
- Project-level (`.claude/hooks/`) vs global (`~/.claude/hooks/`)
- Script language: Bash (default) or Python (if parsing JSON or complex logic)
- What it outputs to Claude vs what the user sees

### If type = `mcp`

1. What external system does this connect to? (e.g., Postgres DB, internal REST API, Jira, Notion, a file system Claude can't reach)
2. Which MCP primitives are needed?
   - **Tools**: model-invoked actions with side effects (e.g., `createIssue`, `queryDatabase`)
   - **Resources**: read-only data addressable by URI (e.g., `db://users/123`)
   - **Prompts**: user-triggered parameterized templates (e.g., `/summarize-sprint`)
3. Is this personal/local (stdio transport, runs as a local process) or shared/remote (HTTP transport, team-wide)?
4. What credentials or secrets does it need? (API keys, DB connection strings, tokens)

Additional context to extract if not covered:
- Language preference: TypeScript (recommended) or Python
- Whether it needs to write/mutate the external system or is read-only
- Team-shared (project `.claude/settings.json`) vs personal (`~/.claude/settings.json`)

---

## Phase 3.5: Existence Check

Before building anything, search for existing solutions that already do what the user described. Use context from Phases 1–3.

### Search targets by type

**MCP servers** — richest ecosystem, always search all three:
- `site:smithery.ai <topic keywords>`
- `site:mcp.so <topic keywords>`
- GitHub: search `"modelcontextprotocol <topic>"` with stars filter
- Also check: `https://modelcontextprotocol.io/registry`

**Skills / Agents:**
- GitHub: search `"claude-code skill <topic>"` OR `"claude-code agent <topic>"`
- GitHub topics: `claude-code-skills`, `claude-code-agents`

**Hooks:**
- GitHub: search `"claude code hook <event-type> <topic>"`
- GitHub topic: `claude-code-hooks`

### How to present findings

If **1–3 strong matches** found, present as a compact table:

```
Found existing solutions:

| Name | What it does | Source / Install |
|------|-------------|-----------------|
| <name> | <one line> | <URL or install command> |
```

Then ask: **"Found existing [type] that match — what would you like to do?"**
Options:
- Use the existing one — provide install/setup instructions and stop
- Use it as a starting point — scaffold from it then customize
- Build from scratch — I need something different

If **no matches found**: state one line `No existing [type] found for this — building from scratch.` and continue.

If **4+ matches** found: surface the top 2 by relevance/stars and note how many others exist.

**Strong match criteria:**
- MCP: covers the same external system AND at least 2 of the same operations
- Skill: same trigger phrase category AND same core action
- Agent: same domain AND same tool set
- Hook: same event AND same blocking/observing intent

---

## Phase 4: Build Refined Brief

Synthesize all collected answers into this structured block. Fill in every field.

```
## REFINED BRIEF

Artifact type: [agent | skill | hook | mcp]
Name: [proposed name — snake_case for agents, kebab-case for skills/hooks/mcp]
One-line purpose: [single sentence — what it does, not what it is]
Trigger: [how/when it activates — user phrase, auto-invoke condition, hook event]
Scope: [global (~/.claude/) | project (.claude/) | worktree-isolated]
Side effects: [yes — list them | no]
Tools required: [comma-separated from: Read, Grep, Edit, Write, Bash, Glob, WebFetch, Agent, mcp__* tools]
Arguments / input: [what $ARGUMENTS contains, or "none"]
Output / deliverable: [conversation text | file written to X | blocks tool call | spawns subagents | etc.]
Edge cases to handle:
- [case 1]
- [case 2]
Related artifacts: [skills/agents/hooks it calls, replaces, or might be confused with]
Additional notes: [model preference, isolation requirements, toggleability, script language, anything else]
```

---

## Phase 5: Confirm and Delegate

Announce:

```
Building a [type] named "[name]".
[One sentence: why this type is the right fit.]
```

Then invoke the appropriate conjure command via the Skill tool, passing the full REFINED BRIEF inline as the args string. The brief must be detailed enough that the conjure command skips intake and builds immediately.

- skill → `Skill(skill: "conjure-skill", args: "<name> - <full refined brief>")`
- agent → `Skill(skill: "conjure-agent", args: "<name> - <full refined brief>")`
- hook → `Skill(skill: "conjure-hook", args: "<full refined brief including event, name, and purpose>")`
- mcp → `Skill(skill: "conjure-mcp", args: "<name> - <full refined brief including external system, primitives, transport, secrets>")`
