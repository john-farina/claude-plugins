---
name: conjure-config
description: Set, view, or remove conjure preferences. Asks questions to understand what you want, then writes plain-language instructions that conjure commands follow automatically.
argument-hint: "[show | remove | reset | <natural language description>]"
allowed-tools: Read,Write,Edit,Bash,AskUserQuestion
user-invocable: true
---

# /conjure-config — Manage Conjure Preferences

Conjure reads preferences from two optional plain-text files. Neither is required — conjure works without them.

- **User-global**: `~/.claude/conjure-config.md` — applies across all projects
- **Project-local**: `.claude/conjure-config.md` in the current working directory — supplements or overrides global for this project

Both are just markdown files with sections of natural-language instructions. Conjure commands read them at startup and follow whatever is written there. You can say anything — output paths, naming rules, required tools, code style constraints, workflow steps, template content. If Claude can understand it, conjure will follow it.

---

## Phase 0: Parse Arguments

| Args | Action |
|---|---|
| *(empty)* | Interactive intake — ask what they want to configure |
| `show` | Print the current config (both files, labeled) |
| `remove <type>` | Remove a specific section (`skills`, `agents`, `hooks`, `mcp`, `global`) |
| `reset` | Clear a config file (confirm scope and confirm deletion) |
| Any other text | Treat as a description of the desired preference — go to Phase 2 |

---

## Phase 1: Show Current Config (if `show`)

Read both config files and print them formatted:

```
~/.claude/conjure-config.md (global)
─────────────────────────────────────
[contents or "(empty)"]

.claude/conjure-config.md (project-local)
─────────────────────────────────────────
[contents or "(not found)"]
```

Done. No further phases needed.

---

## Phase 2: Interactive Intake

If the user gave a description in `$ARGUMENTS`, use it as starting context. If they gave nothing, ask:

> "What do you want conjure to do differently? Describe it however you like — a folder path, a naming rule, a code style requirement, a template, anything."

Then ask targeted follow-up questions via `AskUserQuestion` to get clarity. Use your judgment about what's ambiguous — not every answer needs follow-up. Common gaps to probe:

- **Scope**: does this apply to all conjure commands, or just one type (skills / agents / hooks / mcp)?
- **Always vs. default**: should this always happen, or only when no other instruction applies?
- **Path ambiguity**: if they gave a relative path, confirm the absolute path
- **Condition**: does this apply in all situations, or only when a condition is met?
- **Conflicts**: if this contradicts something already in the config, ask which wins

Do NOT ask about things that are already clear from context.

---

## Phase 3: Scope

Ask which file to write to (if not already obvious from context):

- `Global` — `~/.claude/conjure-config.md` — applies to all projects
- `Project-local` — `.claude/conjure-config.md` — applies only here (never committed)

If project-local: check whether `.gitignore` exists. If it does, check whether `.claude/conjure-config.md` is already ignored. If not, offer to add it.

---

## Phase 4: Write to Config

### File format

The config file is a markdown document with named sections. Each section contains free-form instructions in plain language. No schema, no required fields — just write what conjure should do.

```markdown
---
updated: YYYY-MM-DD
---

## global
[Instructions that apply to all conjure commands]

## skills
[Instructions specific to /conjure-skill]

## agents
[Instructions specific to /conjure-agent]

## hooks
[Instructions specific to /conjure-hook]

## mcp
[Instructions specific to /conjure-mcp]
```

### How to write the instructions

Write them exactly as you would write a rule in CLAUDE.md: direct, imperative, specific. Examples of well-written config instructions:

```
## skills
Always write skill files to ~/.claude/commands/ios/ instead of the default location.
Name skills with the prefix "ios-" (e.g., ios-commit, ios-review).
Always add `keep-coding-instructions: true` to frontmatter.
Always include Bash and Glob in allowed-tools.

## agents
Default model is opus. Never use haiku.
Store agents in ~/.claude/agents/ios/.
Always check ~/triumph-sdk/ios/.claude/docs/ for project context before interviewing the user.

## global
Author is John Farina.
Use gt (Graphite CLI) instead of git for all version control instructions.
When referencing commits in generated skills, always use `gt commit` not `git commit`.
```

### Write rules

1. Read the target file first if it exists
2. Find the matching section (`## global`, `## skills`, etc.)
3. If the section exists: append the new instruction to it (do NOT replace the whole section)
4. If the section doesn't exist: add it at the bottom
5. Update `updated:` date in frontmatter
6. Never wipe unrelated sections

If the new instruction contradicts something already in the section, remove the old contradicting line and add the new one.

---

## Phase 5: Confirm

After writing, show a brief confirmation:

```
✓ Added to ~/.claude/conjure-config.md → ## skills

  "Always write skill files to ~/.claude/commands/ios/ instead of the default location."

Conjure commands will follow this on their next run.
```

---

## Phase 6: Remove / Reset

### `remove <type>`

1. Read the file
2. Find the `## <type>` section and delete everything in it (keep the header or remove it too if empty)
3. Confirm: `✓ Cleared ## <type> from ~/.claude/conjure-config.md`

### `reset`

1. Ask: global, project-local, or both?
2. Confirm deletion: "This will delete all conjure preferences in [file]. Are you sure?"
3. Delete the file
4. Confirm: `✓ Deleted ~/.claude/conjure-config.md — conjure commands will use default behavior`
