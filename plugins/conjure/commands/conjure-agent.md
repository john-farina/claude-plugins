---
name: conjure-agent
description: Creates or updates a Claude Code agent definition (~/.claude/agents/<name>.md or project-local).
allowed-tools: Read,Write,Edit,Bash,Glob
user-invocable: true
argument-hint: "<agent-name> [description|faster|simpler|detailed]"
---

# Craft Agent

Creates or updates a Claude Code agent definition (`~/.claude/agents/<name>.md` or project-local).

---

## Phase 0 — "Should this be an agent at all?"

Before creating anything, apply this gate. An agent is the RIGHT tool only when one of these is true:

1. **Context pollution** — prior subtask results (1000+ tokens) would degrade reasoning on the next step
2. **True parallelism** — independent workstreams that gain nothing from sharing a context window
3. **Tool overload** — the job needs 15+ tools, or fundamentally conflicting behavioral modes (e.g., "never edit" and "always edit")
4. **Sustained specialization** — the agent accumulates domain context across many calls that a generalist can't maintain

If none apply: a well-prompted single Claude call (or a skill/command) achieves equivalent results at 10-15x lower token cost. Say so and stop.

**Redirect table — say this and stop if matched:**

| Request sounds like… | Actually is… | Say… |
|---|---|---|
| "agent to commit my changes" | Skill — user-triggered, inline workflow | "This is better as a skill — run `/conjure-skill <name>`" |
| "agent to run lint whenever I save" | Hook — fires automatically | "This is better as a hook — run `/conjure-hook <description>`" |
| "agent to explain architecture" | Skill (reference) | "This is better as a skill — run `/conjure-skill <name>`" |
| "agent to search the codebase" | Agent ✅ — isolated context, specific tools | Continue |
| "agent to investigate crashes" | Agent ✅ — spawned programmatically | Continue |
| "agent to always do X" | Hook — automatic, event-driven | "This is better as a hook — run `/conjure-hook <description>`" |

If updating an existing agent: skip to Phase 1 — the decision was already made.

---

## Phase 0.5: Load User Preferences

Before doing anything else, read both conjure config files if they exist:

```bash
cat ~/.claude/conjure-config.md 2>/dev/null
cat .claude/conjure-config.md 2>/dev/null
```

Follow all instructions found in `## agents` and `## global` sections throughout every phase of this command. Project-local instructions (`.claude/conjure-config.md`) take precedence over global ones when they conflict. If neither file exists, proceed with defaults.

---

## Phase 1 — Parse arguments and detect mode

Parse `$ARGUMENTS`:

| Pattern | Mode |
|---------|------|
| `<name>` only, file exists | **Update** — read + refine existing agent |
| `<name>` only, file missing | **Create** — ask focused questions then build |
| `<name> faster` | **Speed-optimize** — reduce turns, add pre-injection, trim prose |
| `<name> simpler` | **Simplify** — strip to 80% case, move edge cases to bottom |
| `<name> detailed` | **Expand** — add phases, edge cases, examples |
| Free-form sentence | Infer name (kebab-case) + intent, go to create |

Find the existing file:
```bash
find ~/.claude/agents/ <project-root>/.claude/agents/ \
  -name "<name>.md" 2>/dev/null | head -1
ls ~/.claude/agents/ 2>/dev/null
```

If found: read it fully before proceeding. If transform mode: apply the transform directly — no interview needed.

---

## Phase 2 — Interview (create/update mode only)

Use AskUserQuestion with all questions in one call:

**Q1 "Role"** — "Is this agent an orchestrator (plans + delegates) or a worker (executes one focused job)?"
Options: Worker — executes one focused task · Orchestrator — decomposes work, spawns subagents · Both — single agent doing end-to-end work

**Q2 "Core job"** — "What is the ONE thing this agent does? Be specific."
Options: Code review/analysis · Codebase search/exploration · Build/CI failure diagnosis · File editing/code generation · Research/web lookup · Orchestrate parallel workers

**Q3 "Capabilities"** (multiSelect) — "What does this agent need to do?"
Options: Read files and search codebase · Run shell commands (git, gh, build tools) · Edit or create files · Fetch web pages/GitHub API · Spawn subagents (orchestrator only) · Write outputs to files for handoff

**Q4 "Trigger"** — "When should this agent auto-spawn?"
Options: After any code change · When build/test fails · When I ask to review something · Only when explicitly called by name

**Q5 "Scope"** — "Where should this agent live?"
Options: User-global (~/.claude/agents/) · This project only (<project-root>/.claude/agents/)

---

## Phase 3 — Architecture decisions

### Orchestrator vs. Worker — the most important choice

**Worker agents:**
- Single focused role, matched tool set, clear output format
- Should write outputs to files, not return large summaries through conversation
- 10–20 maxTurns typical
- Never spawn subagents — the `Agent` tool is filtered out of spawned subagents at runtime; never list it in a worker's `tools`

**Orchestrator agents:**
- Plan, decompose, delegate — then synthesize
- Must include explicit scaling rules: "simple = 1–3 subtasks, complex = 5–10 parallel workers"
- Must route subagent outputs through artifact files, not conversation messages
- 40–100 maxTurns; use `opus` model
- Must have `Agent` in tools
- ⚠️ Only works when the agent runs as the main session thread — subagents spawned via `Agent` tool CANNOT spawn further subagents (hard constraint, not configurable)

The "telephone game" failure mode: orchestrators that receive full subagent output as text lose fidelity at every hop. Fix: subagents write to files at a known path; orchestrator reads files directly.

### Coordination pattern — pick one

| Pattern | Use when | Termination |
|---------|----------|-------------|
| **Orchestrator-Worker** | Unpredictable decomposition, parallelizable subtasks | All workers complete or N retries |
| **Generator-Verifier** | Quality-critical output with explicit pass/fail criteria | Verifier passes OR max iterations reached |
| **Agent Teams** | Long-running parallel work that benefits from sustained context | Time budget or explicit convergence check |
| **Router/Triage** | Fan-out to domain specialists based on input classification | Single dispatch — no loop |
| **Shared State** | Agents build on each other's discoveries | Time budget or convergence threshold — REQUIRED, or it loops forever |

For Generator-Verifier: define pass/fail criteria BEFORE running. Vague criteria → rubber-stamping.

### Context passing — treat every agent like a new employee

When an orchestrator spawns a worker, the worker has zero context from the parent conversation. The orchestrator's delegation prompt must include:
- What the task is (specific, bounded)
- What files/data to look at (explicit paths or search commands)
- What output format is expected (exact structure)
- What tools to prefer
- Any project constraints that apply

Never write: "Based on prior context, handle this." The worker has no prior context.

### Context-first pattern — check before asking

Before interviewing the user about domain details, check for a context file:
```bash
ls ~/.claude/docs/<domain>*.md 2>/dev/null
ls <project-root>/.claude/docs/<domain>*.md 2>/dev/null
```
If a relevant context file exists, read it first. Only ask for information not covered or specific to this invocation.

---

## Phase 4 — Decide every frontmatter field

### `name`
- Lowercase + hyphens, max 3 words
- Update mode: keep existing name
- Create mode: derive from core job, check for conflicts with existing agents

### `description` — THE most important field
Structure: `[Specific role]. [Trigger: "Use proactively when X" or "Use immediately after Y"]. [Scope/boundary]. NOT for [disambiguation].`

- ❌ "Reviews code" — too vague, won't auto-invoke correctly
- ✅ "Reviews files against project standards. Use proactively after any code change. NOT for backend code."
- Explicit-only agents: "Use ONLY when explicitly invoked by name."
- Add a "NOT for" clause whenever a similar agent exists — prevents wrong agent from triggering

### `tools`
Always set an explicit list. Never omit. Match tools to actual behavior — agents with poor/wrong tool descriptions fail completely.

| Need | Tools |
|---|---|
| Search codebase | `Read, Grep, Glob` |
| Shell commands | `Bash` |
| Edit files | `Edit, Write` |
| Fetch web/API | `WebFetch` |
| Spawn subagents (orchestrators only) | `Agent` |
| Web research | `WebSearch, WebFetch` |

Read-only agents: add `disallowedTools: Write, Edit` as defense-in-depth.

**`disallowedTools` interaction rule:** `disallowedTools` is applied first, then `tools` resolves against the remainder. A tool listed in both is removed.

**`Agent` tool warning:** Do NOT include `Agent` in a worker subagent's `tools` list — the runtime filters it out. Only orchestrators running as the main session thread can spawn subagents.

**Plugin agents:** Agents installed via plugins do NOT support `hooks`, `mcpServers`, or `permissionMode` fields (security restriction). Omit those fields for plugin-distributed agents.

### `model`
| Task | Model | Alias |
|---|---|---|
| Fast search / classify / triage | `claude-haiku-4-5-20251001` | `haiku` |
| Code review, analysis, debug, worker | `claude-sonnet-4-6` | `sonnet` |
| Architecture, orchestration, planning | `claude-opus-4-7` | `opus` |
| Inherit parent session model | — | `inherit` |

Both full IDs and aliases are accepted. Default is `inherit`.

### `permissionMode`
| Scenario | Value |
|---|---|
| Default — prompts for risky ops | omit or `default` |
| Read-only, must never edit | `plan` |
| Edits files with trust | `acceptEdits` |
| Auto-approve based on session rules | `auto` (requires v2.1.83+, specific plans/models) |
| Skip permission prompts entirely | `dontAsk` |
| Bypass all permission checks | `bypassPermissions` |

**Inheritance rules:** Under a `bypassPermissions` parent, child mode cannot be overridden. Under `auto` mode parent, the subagent's `permissionMode` is ignored entirely.

### `maxTurns` — always set
| Type | Value |
|---|---|
| Simple fetch/format | 5 |
| Code reviewer (read + analyze) | 10 |
| Debugger / diagnostic | 15 |
| Grep-heavy explorer | 20 |
| Multi-phase pipeline | 40 |
| Orchestrator with parallel workers | 80–100 |

### `memory`
Three scopes — pick the right one:
- `memory: user` — cross-project learnings, stored in `~/.claude/agent-memory/<name>/`
- `memory: project` — team-shareable, stored in `.claude/agent-memory/<name>/` (checked in)
- `memory: local` — project-specific personal, stored in `.claude/agent-memory-local/<name>/` (git-ignored)

First 200 lines of `MEMORY.md` from that directory are auto-injected into the agent's system prompt. Omit if stateless.

### `isolation`
Set `isolation: worktree` if agent edits files AND may run in parallel. Omit for serial/read-only.

### `color`
Values accepted by the `/agents` UI (use these exact strings):
`automatic` · `red` · `blue` · `green` · `yellow` · `magenta` · `purple` · `orange` · `pink` · `cyan`

Suggested by role: Review/quality: `red` · Search: `cyan` · Build/CI: `yellow` · Editing: `green` · Research: `blue` · Config: `purple` · Orchestrator: `orange` · Social/misc: `pink`

### Additional fields (optional)

| Field | Purpose |
|---|---|
| `prompt` | Alternative to the frontmatter body — inline system prompt string |
| `skills: [name1, name2]` | Preloads full skill content into subagent context at startup — saves turns, improves first-call performance |
| `mcpServers: [name]` | Scopes MCP servers to this subagent (not supported in plugin agents) |
| `hooks:` | Lifecycle hooks scoped to this subagent's active lifetime (not supported in plugin agents) |
| `background: true` | Run asynchronously — spawns immediately, returns task ID, results retrieved later via TaskOutput |
| `effort: low\|medium\|high\|xhigh\|max` | Override session effort level for this agent |
| `initialPrompt: "..."` | Auto-submitted as first user turn when agent runs as main session agent via `--agent` |

---

## Phase 4.5 — Project Enforcement Layer

<!-- Inject project-specific rules here in your wrapper.
     Example: iOS enforcement, backend constraints, VCS rules, linting standards.
     If no project context file exists, skip this phase. -->

Before writing, check for a project enforcement context file:
```bash
ls ~/.claude/docs/project-constraints*.md 2>/dev/null
ls <project-root>/.claude/docs/*.md 2>/dev/null
```

If found: read it and enforce any rules that apply to the agent being created.

---

## Phase 5 — Write the system prompt body

Structure in this exact order (skip unused sections):

```markdown
## Identity
You are [specific role]. [Specialization]. [One-line philosophy].

## Hard constraints
[3-5 NEVER rules — only non-obvious ones]

## Workflow
1. [First action]
2. [Next action]
[Inline shell commands where useful]

## Output format
[Exact output structure as a code block]

## Domain knowledge
[Project-specific facts only — no general docs]
```

### For orchestrator agents, add:

```markdown
## Orchestration rules
- Scale effort to complexity: simple task = 1–3 workers, complex task = 5–10 parallel workers
- Workers write outputs to: `/tmp/<agent-name>/<subtask-id>.md` — read those files, don't ask workers to summarize in-conversation
- Each worker prompt must be self-contained: explicit task, paths to read, exact output format, relevant constraints
- Set explicit termination: max N iterations OR convergence check — never open-ended loops
- Parallel tool calls within each worker cut execution time ~90% — instruct workers to use 3+ tools simultaneously when possible

## Worker delegation template
When spawning a worker, include:
1. Task (specific, bounded — not "handle this")
2. Files/paths to examine (explicit)
3. Output format (exact structure)
4. Preferred tools
5. Constraints (arch rules, VCS rules, etc.)
```

### For generator-verifier agents, add:

```markdown
## Verification criteria (define before running)
PASS: [explicit conditions that constitute success]
FAIL: [explicit conditions that require regeneration]
MAX_ITERATIONS: [N] — stop after N loops regardless of verdict
```

Anti-patterns:
- ❌ Flattery ("You are an EXTREMELY talented...")
- ❌ Dynamic values (timestamps/dates) that break prompt cache
- ❌ Rigid checklists that eliminate autonomous judgment
- ❌ Docs that can be looked up — project-specific facts only
- ❌ Workers that return large text summaries instead of writing files
- ❌ Orchestrators without explicit termination conditions
- ❌ Worker prompts that say "based on prior context" — workers have no prior context
- ❌ `Agent` in a worker's `tools` list — it's filtered out at runtime

Target: 800–2,000 tokens. Cut lowest-value prose first if over limit.

If `memory: project`: add memory instructions block:
```
## Memory
Before starting: read memory for relevant patterns.
After completing: write key findings — format: FINDING: X | WHERE: file:line | PATTERN: rule
```

---

## Phase 6 — SubagentStop hook (if quality gate requested)

If verdict injection or force-iteration was requested, add a case to `~/.claude/hooks/subagent-stop-gate.sh`:

```bash
  <agent-name>)
    if echo "$LAST_MESSAGE" | grep -q "BLOCKING"; then
      echo '{"decision": "block", "reason": "Blocking issues found — fix before continuing."}'
      exit 2
    elif echo "$LAST_MESSAGE" | grep -q "STAMP"; then
      echo '{"systemMessage": "✓ <agent-name>: approved"}'
    fi
    ;;
```

Read the hook file before editing. Add BEFORE the `*)` default case.

---

## Phase 7 — Write and confirm

Output path:
- User-global: `~/.claude/agents/<name>.md`
- Project: `<project-root>/.claude/agents/<name>.md`

Write the file. Then print:

```
✓ [Created|Updated]: <path>

Name:        <name>
Role:        <orchestrator | worker | end-to-end>
Pattern:     <orchestrator-worker | generator-verifier | router | agent-teams | shared-state | n/a>
Enforcement: <project rules applied / not applicable>
Triggers:    "<first 80 chars of description>"
Model:       <model>
Tools:       <tool list>
maxTurns:    <n>
Memory:      <yes/no>
Isolation:   <worktree / none>
Hook:        <yes/no — what it does>
Token cost:  <note if orchestrator — "~10-15x vs single agent">
```

Update mode: also print a diff summary — `<field>: <old> → <new>` for each changed field.

---

## Quick reference — when to use each pattern

| You want to... | Use |
|---|---|
| Review code quality | Worker (read-only, sonnet, 10 turns) |
| Search codebase for patterns | Worker (read-only, haiku, 20 turns) |
| Fix a complex multi-file bug | End-to-end or Orchestrator-Worker |
| Ensure output meets a standard | Generator-Verifier (define pass/fail first) |
| Route to domain experts | Router (triage agent + specialist workers) |
| Run independent research paths | Orchestrator-Worker (parallel) |
| Fan out to many simultaneous tasks | Agent Teams (sustained context per worker) |
| Collaborative discovery | Shared State (requires time budget termination) |
