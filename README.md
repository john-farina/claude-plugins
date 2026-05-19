# conjure — Build Claude Code Extensions from Plain Language

> Build Claude Code skills, agents, hooks, and MCP servers from plain-language descriptions. Describe what you want — conjure writes production-ready artifacts instantly. No boilerplate. No manual wiring.

```
claude plugin marketplace add john-farina/claude-plugins
claude plugin install conjure@john-farina
```

**[→ conjure.dev landing page](https://john-farina.github.io/claude-plugins/)**

---

## What is conjure?

Conjure is a Claude Code plugin that builds the four types of Claude Code extensions:

- **Skills** — slash commands you type (`/my-command`)
- **Agents** — autonomous workers Claude spawns programmatically
- **Hooks** — event-driven shell scripts that fire automatically
- **MCP Servers** — live connections to external APIs, databases, and tools

Describe what you want in plain language. Conjure classifies the extension type, asks targeted questions, checks whether a published solution already exists, and writes a production-ready artifact. Every command also handles updates — no need to say "update."

---

## How to build a Claude Code skill

```
/conjure-skill "summarize my git changes and write a commit message in my team's style"
```

Conjure writes the `.md` command file with proper frontmatter, tool declarations, argument handling, and phased instructions. Immediately usable in the same session.

---

## How to build an MCP server for Claude Code

```
/conjure-mcp "connect Claude to our internal Postgres database"
```

Conjure searches smithery.ai and mcp.so first. If no match exists, it scaffolds a full server in Node.js, Python, or Go with typed tools and registers it in `settings.json`.

---

## How to create a Claude Code agent

```
/conjure-agent "review PRs and post feedback in my voice"
```

Conjure handles architecture decisions (orchestrator vs. worker), tool selection, `maxTurns`, `permissionMode`, and model config. Writes the definition to `~/.claude/agents/`.

---

## How to add a Claude Code hook

```
/conjure-hook "run prettier on every file Claude edits"
```

Conjure writes the shell script and the `settings.json` entry. Supports all hook events: `PreToolUse`, `PostToolUse`, `SessionStart`, `UserPromptSubmit`, `Stop`.

---

## Commands

| Command | What it does |
|---|---|
| `/conjure` | Auto-detects type from description and delegates |
| `/conjure-skill` | Create or update a slash command / workflow |
| `/conjure-agent` | Create or update an autonomous worker |
| `/conjure-hook` | Create or update an event-driven hook |
| `/conjure-mcp` | Create or update an MCP server |
| `/conjure-config` | Set preferences all conjure commands follow automatically |
| `/conjure-help` | Full reference and decision guide |

---

## conjure-config — preferences that apply everywhere

Run `/conjure-config` once. Tell conjure your output paths, VCS tool, author name, naming conventions, required frontmatter — anything. It writes plain-language instructions to `~/.claude/conjure-config.md` (global) and `.claude/conjure-config.md` (project-local). Every conjure command reads these automatically and skips questions you've already answered.

---

## License

MIT
