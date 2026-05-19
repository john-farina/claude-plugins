---
name: conjure-help
description: Show help for the conjure toolkit — all commands, argument syntax, update modes, and decision guide. Use when you want to know which conjure command to use or how any conjure command works.
argument-hint: "[command-name]"
allowed-tools: Read
user-invocable: true
---

## Conjure — Help

Conjure builds Claude Code extensions: skills (slash commands), agents (autonomous workers), hooks (event-driven scripts), and MCP servers (external connections). All commands support both **create** and **update** modes.

---

## Which command do I use?

```
Does Claude need live access to an external system (DB, API, third-party service)?
  YES → /conjure-mcp

Should it fire automatically on an event (file edit, tool call, session start)?
  YES → /conjure-hook

Does Claude spawn it with its own context, model, or restricted tools?
  YES → /conjure-agent

Does the user type a /command to trigger it?
  YES → /conjure-skill

Not sure? → /conjure — auto-detects the type for you
```

**Signal phrases:**

| You say… | Use |
|---|---|
| "connect to", "integrate with", "query [DB/API]", "stop copy-pasting from [service]" | `/conjure-mcp` |
| "whenever", "every time", "fires when", "on save", "after tool call", "session start" | `/conjure-hook` |
| "investigate", "search in isolation", "spawn a worker", "orchestrate" | `/conjure-agent` |
| "command", "slash command", "workflow I trigger", "checklist" | `/conjure-skill` |

---

## Command Reference

### `/conjure` — Unified entry point
Auto-detects type, runs intake, delegates to the right command.

```
/conjure                              Ask what you want to build
/conjure <description>                Auto-classify from description
/conjure help                         Show this help
```

---

### `/conjure-skill` — Slash commands & workflows

```
/conjure-skill <name>                 Create (if new) or update (if exists)
/conjure-skill <name> faster          Speed-optimize: add ! pre-injection, strip prose
/conjure-skill <name> simpler         Reduce to 80% case, move edge cases to bottom
/conjure-skill <name> detailed        Expand: add phases, edge cases, examples
/conjure-skill <name> blank           Keep frontmatter, clear body
/conjure-skill <name> review          Quality gate: report issues + fix them
/conjure-skill <description>          Create with inline description — builds immediately
```

**Storage:** action skills → `~/.claude/commands/<name>.md` · knowledge skills → `~/.claude/skills/<name>/SKILL.md`

---

### `/conjure-agent` — Autonomous workers & orchestrators

```
/conjure-agent <name>                 Create (if new) or update (if exists)
/conjure-agent <name> faster          Reduce turns, add pre-injection, trim prose
/conjure-agent <name> simpler         Strip to 80% case, move edge cases down
/conjure-agent <name> detailed        Add phases, edge cases, delegation template
/conjure-agent <description>          Create from description — infers name
```

**Storage:** `~/.claude/agents/<name>.md` (global) or `.claude/agents/<name>.md` (project)

---

### `/conjure-hook` — Event-driven scripts

```
/conjure-hook <description>           Create new hook from description
/conjure-hook <slug>                  Update existing hook (found by slug in settings.json)
/conjure-hook <slug> review           Audit script + settings entry, fix issues
/conjure-hook <slug> add-event        Add a new event to an existing hook script
/conjure-hook help                    Show argument table
```

**Storage:** script at `~/.claude/hooks/<name>.sh` + entry in `~/.claude/settings.json`

**Events (common):**

| Event | Blocks? | Use for |
|---|---|---|
| `PreToolUse` | YES (exit 2) | Security guards, pattern injection |
| `PostToolUse` | No | Lint, format, log |
| `SessionStart` | No | Load context, set env vars |
| `UserPromptSubmit` | YES (exit 2, max 30s) | Inject context on every prompt |
| `Stop` | Can continue | Quality gates, notifications |

---

### `/conjure-mcp` — MCP servers (external connections)

```
/conjure-mcp <description>            Create new MCP server
/conjure-mcp <name>                   Update existing server (found by directory/settings)
/conjure-mcp <name> add-tool          Add a new tool to an existing server
/conjure-mcp <name> add-resource      Add a new resource to an existing server
/conjure-mcp <name> review            Audit server code + registration, fix issues
/conjure-mcp help                     Show argument table
```

**Storage:** `~/.claude/mcp-servers/<name>/index.js` + entry in `~/.claude/settings.json` `mcpServers`

**MCP primitives:**

| Primitive | Controlled by | Use for |
|---|---|---|
| Tools | Model | Actions with side effects (`createIssue`, `queryDB`) |
| Resources | Application | Read-only data by URI (`db://users/123`) |
| Prompts | User | Slash command templates (`/summarize-sprint`) |

---

### `/conjure-config` — User preferences

Set instructions that every conjure command follows automatically. Anything goes — output paths, naming rules, required frontmatter, code style, templates, VCS tool, author name. Stored in `~/.claude/conjure-config.md` (global) or `.claude/conjure-config.md` (project-local).

```
/conjure-config                       Interactive — ask what to configure
/conjure-config show                  Print current preferences (both files)
/conjure-config remove skills         Remove all skills-specific preferences
/conjure-config reset                 Clear a config file entirely
```

---

## Update mode — how it works for all commands

Every conjure command auto-detects whether you're creating or updating:

1. Parses `$ARGUMENTS` for a name/slug
2. Searches for the existing artifact (file, settings entry, or server directory)
3. If found → **reads it fully first**, then refines/updates
4. If not found → **creates new**

You never need to say "update" — passing the name of something that exists always triggers update mode.

---

## Quick examples

```
/conjure I want Claude to lint my Python files after every edit
→ /conjure detects: hook (fires after PostToolUse on .py files)

/conjure make a command that helps me write commit messages
→ /conjure detects: skill (user-triggered workflow)

/conjure connect Claude to our internal Jira instance
→ /conjure detects: MCP server

/conjure an agent that investigates iOS crash reports
→ /conjure detects: agent (isolated, spawned programmatically)

/conjure-skill gt-commit faster
→ speed-optimize the existing gt-commit skill

/conjure-mcp jira-server add-tool
→ add a new tool to the existing jira-server MCP server
```
