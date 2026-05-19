---
name: conjure-mcp
description: Create a Claude Code MCP (Model Context Protocol) server. Provide a description of what external system or data source you want Claude to connect to.
argument-hint: "<description of what external system to connect or what tools to expose>"
allowed-tools: Read,Write,Edit,Bash
---

## Phase 0: Route the Request

Before building anything, verify an MCP server is the right artifact.

| Question | If YES → use instead |
|---|---|
| Does Claude already have access to this data/system via native tools? | **Skill** → run `/conjure-skill <name>` |
| Is this a repeatable workflow or checklist, not a live connection? | **Skill** → run `/conjure-skill <name>` |
| Should this fire automatically on a tool call event? | **Hook** → run `/conjure-hook <description>` |
| Does Claude spawn this with isolated context? | **Agent** → run `/conjure-agent <description>` |
| Does this require a live, typed, schema-validated connection to an external system? | **MCP** → continue ✅ |

**MCP signal phrases — if you hear any of these, it's an MCP:**

| Request sounds like… | Actually is… |
|---|---|
| "connect to / integrate with / access [external system]" | MCP ✅ |
| "query our database / hit our REST API / read from Postgres" | MCP ✅ |
| "stop copy-pasting from Jira/Linear/Figma/Slack" | MCP ✅ |
| "Claude needs to call [external service]" | MCP ✅ |
| "hook that fetches from our API" | MCP — hooks can't make authenticated external calls reliably |
| "agent to query the database" | MCP — agents use tools; the tool should be an MCP server |
| "skill to read from Notion" | MCP — skills are procedural, not connective |

**Cost trade-off:** Each MCP server with ~10 tools consumes ~10,000 tokens at session start (always). A skill costs ~100 tokens until activated. Only use MCP when the external access is genuinely required.

---

## Phase 0.5: Load User Preferences

Before doing anything else, read both conjure config files if they exist:

```bash
cat ~/.claude/conjure-config.md 2>/dev/null
cat .claude/conjure-config.md 2>/dev/null
```

Follow all instructions found in `## mcp` and `## global` sections throughout every phase of this command. Project-local instructions (`.claude/conjure-config.md`) take precedence over global ones when they conflict. If neither file exists, proceed with defaults.

---

## Phase 1: Parse Arguments + Detect Mode

Parse `$ARGUMENTS`:

| Pattern | Mode |
|---------|------|
| empty | **Create** — ask what external system to connect |
| description only (no name slug) | **Create** — derive name from description |
| `<name>` only, server directory exists | **Update** — read server files + settings entry, refine |
| `<name>` only, not found | **Create** — use name as the server name |
| `<name> add-tool` | **Extend** — add a new tool to an existing server |
| `<name> add-resource` | **Extend** — add a new resource to an existing server |
| `<name> review` | **Review** — audit server code + registration, report issues |
| `help` | Stop and print the Phase 1 argument table above, then stop |

**Finding an existing server by name:**
```bash
# Check for server directory
ls ~/.claude/mcp-servers/<name>/ 2>/dev/null
ls .claude/mcp-servers/<name>/ 2>/dev/null

# Check if registered in settings
python3 -c "
import json
s = json.load(open('${HOME}/.claude/settings.json'))
servers = s.get('mcpServers', {})
print(list(servers.keys()))
" 2>/dev/null
```

**If Update mode:** Read `index.js` (or `server.py`) and the settings.json `mcpServers` entry fully before proceeding to Phase 3.

**If Review mode:** Read both, then report:
- Are all tools properly typed with Zod schemas?
- Are secrets in env vars (not hardcoded)?
- Is the server registered in settings.json?
- Does `claude mcp test <name>` pass?
- Apply fixes, then output the Phase 5 report.

---

## Phase 2: Understand

The user's intent is: **$ARGUMENTS** (or clarified via intake)

If creating and intent is still unclear, ask what external system or capability to expose.

Determine:
- What external system or data source needs to be connected?
- Which MCP primitives are needed: **Tools**, **Resources**, and/or **Prompts**?
- What transport: **stdio** (local CLI process) or **HTTP** (remote/shared server)?
- User-level (`~/.claude/`) or project-level (`.claude/settings.json`)?
- Language: **TypeScript** (recommended, best SDK support) or **Python**?

## Phase 3: Design

### Choose the right MCP primitives

| Primitive | Controlled by | When to use | Examples |
|---|---|---|---|
| **Tools** | Model | Model decides when to invoke; may have side effects | `searchIssues()`, `createEvent()`, `sendMessage()` |
| **Resources** | Application (host) | Read-only data sources identified by URI; context injection | `file:///report.pdf`, `db://users/123`, `jira://issues/PROJ-42` |
| **Prompts** | User | Reusable parameterized templates surfaced as slash commands | `/summarize-sprint`, `/plan-feature` |

**Default:** most servers expose **Tools** only. Add Resources when the data is read-only and addressable by URI. Add Prompts when users would benefit from parameterized slash commands that call your server.

### Choose transport

| Transport | Use when | Pros |
|---|---|---|
| **stdio** | Local process on same machine, personal tooling | Zero infra, simple setup, launched by Claude Code directly |
| **Streamable HTTP** | Remote server, team-shared, cloud-hosted | Multi-client, persistent, no local install required |

Default to **stdio** for local personal MCP servers.

## Phase 4: Execute

### 1. Scaffold the server

**TypeScript (recommended):**

```bash
mkdir -p ~/.claude/mcp-servers/<name>
cd ~/.claude/mcp-servers/<name>
npm init -y
npm install @modelcontextprotocol/sdk zod
```

Set in `package.json`:
```json
{
  "type": "module",
  "bin": { "<name>": "./index.js" },
  "scripts": { "build": "tsc" }
}
```

**Python alternative:**
```bash
pip install mcp
```

### 2. Write the server

**TypeScript template:**

```typescript
#!/usr/bin/env node
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({
  name: "<name>",
  version: "1.0.0",
});

// Tool — model decides when to call this
server.tool(
  "tool-name",
  "One-line description of what this tool does",
  {
    param: z.string().describe("What this parameter is for"),
  },
  async ({ param }) => {
    // Call external system here
    return {
      content: [{ type: "text", text: `Result: ${param}` }],
    };
  }
);

// Resource — host-controlled read-only data (add only if needed)
// server.resource("resource-name", "scheme://path/{id}", async (uri) => {
//   return { contents: [{ uri: uri.href, text: "data" }] };
// });

// Prompt — user-controlled template (add only if needed)
// server.prompt("prompt-name", { param: z.string() }, ({ param }) => ({
//   messages: [{ role: "user", content: { type: "text", text: `Do X with ${param}` } }],
// }));

const transport = new StdioServerTransport();
await server.connect(transport);
```

Make executable:
```bash
chmod +x ~/.claude/mcp-servers/<name>/index.js
```

Debug before registering:
```bash
npx @modelcontextprotocol/inspector ~/.claude/mcp-servers/<name>/index.js
```

### 3. Register in settings.json

**User-global** (`~/.claude/settings.json`):
```json
{
  "mcpServers": {
    "<name>": {
      "command": "node",
      "args": ["${HOME}/.claude/mcp-servers/<name>/index.js"],
      "env": {
        "API_KEY": "your-key-here"
      }
    }
  }
}
```

**Project-level** (`.claude/settings.json` — shared with team):
```json
{
  "mcpServers": {
    "<name>": {
      "command": "node",
      "args": ["${CLAUDE_PROJECT_DIR}/.claude/mcp-servers/<name>/index.js"]
    }
  }
}
```

**Verify:**
```bash
claude mcp list
claude mcp test <name>
```

## Phase 5: Report

Output exactly:
```
server: <name>
transport: <stdio | http>
primitives: <Tools | Resources | Prompts | combination>
script: <path to server entry point>
registered: <user-global | project-level>
tools: <list of tool names>
does: <one-line description of what the server exposes>
```
