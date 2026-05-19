# claude-plugins

Claude Code plugins I've built and use daily — tools for building extensions, automating workflows, and making Claude smarter about your project.

## Install the marketplace

```
claude plugin marketplace add john-farina/claude-plugins
```

Then install any plugin:

```
claude plugin install conjure@john-farina
```

---

## Plugins

### conjure

> Build Claude Code extensions from plain language.

Describe what you want in plain English — conjure classifies the type (skill, agent, hook, or MCP server), interviews you for the details it needs, searches whether a published solution already exists, and writes a production-ready artifact. No boilerplate. No manual wiring.

**Before conjure:**
```
/conjure-skill → write 80-line command file manually → configure allowed-tools →
test → fix argument handling → realize you need a hook too → start over
```

**After conjure:**
```
/conjure "remind me to run swiftformat before every commit"
→ Done. Hook written, tested, installed.
```

#### What it builds

| Type | What it is | Command |
|---|---|---|
| Skill | A slash command Claude executes (`/my-command`) | `/conjure-skill` |
| Agent | An autonomous worker spawned via the Agent tool | `/conjure-agent` |
| Hook | An event-driven trigger (pre-tool, post-tool, session-start, etc.) | `/conjure-hook` |
| MCP server | A local server that gives Claude new tools via the Model Context Protocol | `/conjure-mcp` |

#### Commands

| Command | What it does |
|---|---|
| `/conjure` | Start here — auto-detects type and delegates |
| `/conjure-skill` | Create or update a slash command / workflow |
| `/conjure-agent` | Create or update an autonomous worker |
| `/conjure-hook` | Create or update an event-driven hook |
| `/conjure-mcp` | Create or update an MCP server |
| `/conjure-config` | Set preferences all conjure commands follow automatically |
| `/conjure-help` | Full reference and decision guide |

#### vs. alternatives

| Approach | Effort | Quality | Conjure |
|---|---|---|---|
| Write manually | High — need to know format, tools, phases | Varies | ✅ Generated from description |
| Copy an example | Medium — adapt to your use case | Often wrong shape | ✅ Purpose-built for your goal |
| `plugin-dev` plugin | Medium — code-first, not natural-language | Good for devs | ✅ Plain language, no boilerplate |
| Ask Claude in chat | Low — but output isn't installed anywhere | One-shot, not persisted | ✅ Written, saved, usable immediately |

```
claude plugin install conjure@john-farina
```

---

## License

MIT
