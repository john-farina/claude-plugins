---
name: conjure-skill
description: "Create or update a Claude Code skill (SKILL.md) with best practices. Usage: /conjure-skill <name> to create or update, /conjure-skill <name> faster|simpler|detailed|blank to transform. Classifies type, picks correct storage location, migrates if misplaced."
allowed-tools: Read,Write,Edit,Bash,Glob
user-invocable: true
argument-hint: "<skill-name> [faster|simpler|detailed|blank] OR <description of new skill>"
---

# /conjure-skill — Create or Update Claude Code Skills

You produce production-quality `.md` skill files. Every skill you write ships immediately — no follow-up steps required.

## Phase 0: Route the Request

Before building anything, verify a skill is the right artifact.

| Question | If YES → use instead |
|---|---|
| Should this fire automatically on an event (file edit, tool call, session start)? | **Hook** → run `/conjure-hook <description>` |
| Does Claude spawn this programmatically with isolated context, a different model, or specific tool restrictions? | **Agent** → run `/conjure-agent <description>` |
| Does it run inline in the main conversation when the user invokes `/command`? | **Skill** → continue ✅ |

**Common mislabels:**

| Request sounds like… | Actually is… |
|---|---|
| "skill to run lint whenever I edit a file" | Hook — fires automatically |
| "skill to search the codebase with haiku model" | Agent — isolated context, specific model |
| "skill to investigate crashes" | Agent — long-running, spawned programmatically |
| "skill to commit my changes" | Skill ✅ — user-triggered, inline workflow |
| "skill that explains architecture" | Skill ✅ — reference, user loads it |

If it's clearly a hook or agent, stop and say: `"This is better as a [hook/agent] — run /conjure-hook or /conjure-agent instead."`

---

## Phase 0.5: Load User Preferences

Before doing anything else, read both conjure config files if they exist:

```bash
cat ~/.claude/conjure-config.md 2>/dev/null
cat .claude/conjure-config.md 2>/dev/null
```

Follow all instructions found in `## skills` and `## global` sections throughout every phase of this command. Project-local instructions (`.claude/conjure-config.md`) take precedence over global ones when they conflict. If neither file exists, proceed with defaults.

---

## Phase 1: Parse Arguments

Parse `$ARGUMENTS`:

| Pattern | Mode |
|---------|------|
| `<name>` only, file exists | **Update** — read + refine existing skill |
| `<name>` only, file missing | **Create** — ask one clarifying question then build |
| `<name> faster` | **Speed-optimize** existing skill |
| `<name> simpler` | **Simplify** existing skill — strip complexity, same core behavior |
| `<name> detailed` | **Expand** existing skill — add phases, sub-agents, references |
| `<name> blank` | **Clear** body, keep frontmatter, reset instructions |
| `<name> - <description>` | **Create** with inline description — build immediately, no question |
| Free-form sentence | Infer name (kebab-case) + description, create immediately |
| `<name> --git-history` | **Git-extract** — analyze git history for pattern extraction, then run quality gate before writing final file |
| `<name> review` | **Review** — run quality gate on existing skill, report issues, apply fixes |

**Storage — one file, one name, never a `commands/*.md` file.** Commands are skills now: a thin alias
command registers the name twice and doubles its listing cost.

Pick the scope first (ask only if the request leaves it open):

| Scope | Path | Use for |
|---|---|---|
| **core** (every session) | `~/.claude/skills/<name>/SKILL.md` | used across repos, weekly or more |
| **repo** | `<repo>/.claude/skills/<name>/SKILL.md`, committed | only meaningful in one repo. Untracked repo skills are invisible to worktree sessions, so commit them |
| **team** | `/triumph-skills:share-skill <name>` after building it under `~/.claude/skills` | everyone on the team |

Rarely used personal tools stay core but get `"<name>": "name-only"` under `skillOverrides` in
`~/.claude/settings.json`, so they're invocable by name without paying listing cost.

**Before writing, check for collisions and prior art:**
```bash
ls ~/.claude/skills .claude/skills 2>/dev/null | grep -i "<name>"
~/.claude/skill-archive/archive.py list <keywords>        # archived skills, restorable by name
PATH=/opt/homebrew/bin:$HOME/.bun/bin:$PATH qmd search "<what it does>" -c skill-archive -n 5
```
If an archived skill fits, restore it (`archive.py restore <name>`) and update it instead of creating a new one.

**Existing file in the wrong place** (a `commands/<name>.md`, or a SKILL.md plus an alias command):
1. Read the existing file(s)
2. Write `~/.claude/skills/<name>/SKILL.md` (or the repo path)
3. Archive the old file: `~/.claude/skill-archive/archive.py move <old-path>`, never `rm`
4. Report the move: `moved: <old-path> → <new-path>`

## Phase 2: Classify Skill Type

Before writing anything, determine the skill's type based on its purpose:

| Type | Characteristics | Use when |
|------|----------------|----------|
| **fast** | ≤30 lines, single action, pre-injected data | Commit, format, submit, one-shot CLI operation |
| **reference** | Inline static content, no tool calls | Conventions, patterns, API docs, lookup tables |
| **standard** | 50–200 lines, clear phases, `$ARGUMENTS` | Feature workflows, multi-step tasks, analysis |
| **comprehensive** | 200–500 lines + supporting files | Complex multi-domain research or orchestration |

Default to **standard** when unsure. Never go comprehensive when standard would work.

## Phase 3: Build Frontmatter

Construct the frontmatter block using only fields that apply:

```yaml
---
name: <kebab-case-name>
description: <trigger-phrase first>. <what it does>. <secondary use cases>.
when_to_use: "<example phrases users would type — counts toward 1,536-char cap with description>"
# --- conditional fields below ---
argument-hint: "[arg1] [arg2]"          # cosmetic — shows in autocomplete only
arguments: [name, type]                  # enables $name substitution in body (positional)
allowed-tools: Read Write Edit Bash Glob # space-separated; GRANTS approval, does NOT restrict
user-invocable: false                    # hides from /menu only; does NOT block Skill tool invocation
effort: low|medium|high|xhigh|max       # overrides session effort level for this skill's turn
context: fork                            # run in isolated subagent context — no conversation history
agent: Explore                           # only with context:fork; options: Explore, Plan, general-purpose, custom agent name
model: haiku|sonnet|opus|inherit         # overrides model for this skill's turn, reverts after
paths: ["src/**/*.ts"]                   # auto-activates skill when matching files are in context
shell: bash                              # bash (default) or powershell
hooks:                                   # lifecycle hooks scoped to this skill's active lifetime
  - event: PostToolUse
    command: "..."
---
```

**Rules:**
- `description` + `when_to_use`: aim for **≤ 300 chars** combined (hard cap 1,536; the agentskills spec caps description at 1,024). Every char is paid in every session's skill listing, and when the listing overflows, Claude Code drops descriptions of the least-used skills first.
- Never add `disable-model-invocation: true` to the user's own skills: agents (farina-bot, loops, subagents) invoke them through the Skill tool, and the flag blocks that. Guard side effects inside the body (a confirm step) instead.
- `user-invocable: false` hides from the `/` menu only. To block programmatic invocation, use `disable-model-invocation: true`.
- `context: fork` creates an isolated subagent with no conversation history. **CLAUDE.md is skipped only when `agent: Explore` or `agent: Plan`** — other agents still load it.
- `allowed-tools` **GRANTS approval** (Claude can use those tools without prompting) — it does NOT restrict. Claude can still call other tools. Scoped syntax: `Bash(git add *)` limits Bash to that pattern.
- Keep `allowed-tools` minimal — list space-separated, only what the skill's body actively calls.
- `arguments` declares named positional params for `$name` substitution. With `arguments: [issue, branch]`, `$issue` = first arg, `$branch` = second.

**Argument substitution variables:**
- `$ARGUMENTS` — full argument string
- `$ARGUMENTS[N]` or `$N` — 0-indexed positional (⚠️ `$0` = first arg, `$1` = second)
- `$name` — named argument (requires `arguments:` frontmatter declaration)
- `${CLAUDE_SKILL_DIR}` — path to the skill's directory (use for portable script paths)
- `${CLAUDE_SESSION_ID}` — current session ID
- `${CLAUDE_EFFORT}` — current effort level

**CSO Rule — description = trigger conditions ONLY, never workflow summary:**
When description summarizes the skill's workflow, Claude short-circuits and follows the description instead of reading the skill body. This silently breaks multi-step skills.
```
❌ "Use when committing — stages files, derives message from diff, runs commit"
✅ "Use when creating a git commit"
❌ "Runs baseline, writes skill, tests with subagent, closes loopholes"
✅ "Use when building or updating a Claude Code skill or command file"
```
Load in keywords Claude would search for: error messages, tool names, symptoms, synonyms. Never describe the process.

## Phase 4: Build Body by Type

### Fast Skill Body Pattern

Pre-inject runtime data with `!` so Claude doesn't need tool calls:

```markdown
!`git diff HEAD --stat`
!`git status --short`

Stage all changes and commit. Derive the message from the diff above. Keep it under 72 chars.
```

Multi-line pre-injection (runs shell block before Claude reads the skill):
````markdown
```!
git log --oneline -10
git status --short
```
````

Rules:
- Under 30 lines
- `!` backtick commands run before Claude sees the skill — use them for git status, file lists, env vars
- State the single action directly — no phases, no conditions, no explanation
- If the action is destructive or irreversible: add one confirmation line

### Reference Skill Body Pattern

```markdown
# <Topic> — Quick Reference

## Key Rules
- Rule 1
- Rule 2

## Patterns

**Pattern name:**
\`\`\`
// example
\`\`\`

## Anti-Patterns
- ❌ Wrong way — why
- ✅ Right way
```

Rules:
- All content inline — no tool calls
- Present tense, imperative where applicable
- Organize by how Claude will look things up (task → rule, not category → rule)

### Standard Skill Body Pattern

```markdown
## Phase 1: <Gather/Detect>

Parse `$ARGUMENTS`. Determine: [what decisions this phase makes].

If `$ARGUMENTS` is empty: [fallback behavior — ask one question OR use sensible default].

\`\`\`bash
# Example pre-flight
git status --short
\`\`\`

## Phase 2: <Execute>

[Action steps — imperative, specific, no narration]

## Phase 3: <Verify/Report>

Confirm the result. Output format:

\`\`\`
[Expected output structure]
\`\`\`

## Proactive Triggers

Surface these without being asked when you notice the condition in context:
- When <condition>: flag <issue>
- When <condition>: recommend <action>

## Related Skills

- **skill-name**: Use when <specific scenario>. NOT for <disambiguation>.
- **skill-name**: Use when <specific scenario>. NOT for <disambiguation>.
```

Rules:
- 2–4 phases maximum
- Each phase has a single clear purpose stated in its heading
- No prose explaining WHY — only WHAT to do
- `$ARGUMENTS` handling explicit in Phase 1
- Bash blocks show exact commands where helpful

### Comprehensive Skill Body Pattern

For skills that genuinely need depth: write to `~/.claude/skills/<name>/` with supporting files.

```
~/.claude/skills/<name>/
├── SKILL.md          ← overview + navigation (under 200 lines)
└── references/
    ├── architecture.md
    ├── patterns.md
    └── examples.md
```

SKILL.md structure:
```markdown
## Overview

[2-3 sentences on what this skill covers and when to use it]

## Quick Start

[The 80% case — what most invocations do]

## Full Workflow

### Phase 1: ...
### Phase 2: ...
### Phase 3: ...

## Reference

- Architecture details: [references/architecture.md](references/architecture.md)
- Patterns: [references/patterns.md](references/patterns.md)
```

## Phase 5: Transformations (update modes)

When transforming an existing skill:

### `faster`
1. Read the existing skill
2. Add `!` pre-injection for any data Claude currently reads via tool calls
3. Add `effort: low` if missing
4. Strip explanatory prose — keep only action statements
5. Collapse multi-step phases into single direct instructions where possible
6. Do NOT remove any behavior — only remove the overhead around it

### `simpler`
1. Read the existing skill
2. Identify the 80% use case
3. Rewrite body to handle only that case in the minimum lines
4. Move edge-case handling to a "## Edge Cases" section at the bottom
5. Remove phases if 2 or fewer steps remain

### `detailed`
1. Read the existing skill
2. Expand into explicit phases if not already present
3. Add `## Edge Cases` section
4. Add `## Examples` section with 2-3 concrete invocation examples
5. If body exceeds 300 lines, convert to directory structure and extract reference content

### `blank`
1. Read frontmatter only
2. Write new file: keep all frontmatter fields, replace body with:
```markdown
<!-- Instructions cleared. Describe the new behavior here. -->
```

## Phase 6: Write the File

Write the complete skill file. Then print:

```
✓ wrote: <path>
  type: <fast|reference|standard|comprehensive>
  invoke: /<name>
  scope: <core|name-only|repo|team>
```

## Phase 6.5: Quality Review (runs after every create, update, or explicit `review` mode)

After writing (or when `review` mode is invoked directly):

Run a quality gate on the output file if one is configured for this project. Check for: CSO compliance in description, token budget, `$ARGUMENTS` handling, proactive triggers, and related-skill disambiguation.

Apply fixes for any warnings. Then evaluate manually:

**Check these in order:**
1. **Description CSO** — trigger conditions only? No workflow summary?
2. **Token budget** — body under 500 lines? Heavy reference content in `references/`?
3. **`$ARGUMENTS` handled** — Phase 1 explicitly parses it?
4. **Proactive triggers present** — does the skill surface relevant issues without being asked?
5. **Related skills present** — disambiguation clause if similar skills exist?
6. **One copy** — `ls ~/.claude/commands/<name>.md` must not exist; no second skill with the same name anywhere in scope.
7. **Description ≤ 300 chars**, frontmatter parses (`uv run --with pyyaml python3 -c "import yaml,sys; yaml.safe_load(open(sys.argv[1]).read().split('---')[1])" <path>`).
8. **Trigger test** — name 2 phrases the user would type; the description must contain their key words. For a team skill, add them as evals via share-skill.

For `review` mode: report findings as a numbered list with severity (fix / warning / suggestion), then apply all fixes and re-run the review script to confirm clean.

For create/update: apply fixes silently before writing the final file. Only surface issues that require a judgment call.

---

## Best Practice Cheat Sheet (apply to every skill you write)

**Frontmatter**
- Trigger phrase in sentence 1 of `description` — not buried
- never `disable-model-invocation` on the user's skills (agents invoke them)
- `allowed-tools` = only what the body actually calls (space-separated)
- Skip optional fields that don't change behavior

**Body**
- Under 500 lines total. Offload reference content to supporting files.
- Imperative mood throughout: "Read the diff. Stage the files. Commit."
- No comments restating what the step name says
- `$ARGUMENTS` parsed explicitly in the first phase
- `!` pre-injection over tool calls for static/predictable data
- One decision per phase heading
- Use `${CLAUDE_SKILL_DIR}` for portable script references inside supporting files

**Anti-patterns to avoid**
- Long narrative explaining why a step exists
- Reading files Claude doesn't need
- `context: fork` on a skill that references conversation state
- writing a `commands/<name>.md` alias next to a SKILL.md
- Phases with multiple unrelated actions
- Body that documents the codebase instead of instructing Claude
- Using `$1` to mean "first arg" — `$0` is first, `$1` is second (0-indexed)
