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

Describe what you want — conjure classifies the type, interviews you for details, checks whether a published solution already exists, and writes a production-ready artifact. Covers all four Claude Code extension types.

| Command | What it does |
|---|---|
| `/conjure` | Start here — auto-detects type and delegates |
| `/conjure-skill` | Create or update a slash command / workflow |
| `/conjure-agent` | Create or update an autonomous worker |
| `/conjure-hook` | Create or update an event-driven hook |
| `/conjure-mcp` | Create or update an MCP server |
| `/conjure-config` | Set preferences all conjure commands follow |
| `/conjure-help` | Full reference and decision guide |

```
claude plugin install conjure@john-farina
```

---

## License

MIT
