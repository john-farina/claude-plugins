---
name: conjure-hook
description: Create or update a Claude Code hook (settings.json entry + shell script). Provide a description of what you want the hook to do.
argument-hint: "<description of what the hook should do>"
allowed-tools: Read,Write,Edit,Bash
---

## Phase 0: Route the Request

Before building anything, verify a hook is the right artifact.

| Question | If YES → use instead |
|---|---|
| Does the user trigger this explicitly via `/command`? | **Skill** → run `/conjure-skill <name>` |
| Does Claude spawn this with isolated context or a different model? | **Agent** → run `/conjure-agent <description>` |
| Should this fire automatically on an event without the user asking? | **Hook** → continue ✅ |

**Common mislabels:**

| Request sounds like… | Actually is… |
|---|---|
| "hook to commit my changes" | Skill — user-triggered |
| "hook to search the codebase" | Agent — isolated, spawned by Claude |
| "hook that fires whenever I edit a file" | Hook ✅ |
| "hook that runs on session start" | Hook ✅ |
| "hook to review my PR" | Skill — user-triggered workflow |

If it's clearly a skill or agent, stop and say: `"This is better as a [skill/agent] — run /conjure-skill or /conjure-agent instead."`

---

## Phase 0.5: Load User Preferences

Before doing anything else, read both conjure config files if they exist:

```bash
cat ~/.claude/conjure-config.md 2>/dev/null
cat .claude/conjure-config.md 2>/dev/null
```

Follow all instructions found in `## hooks` and `## global` sections throughout every phase of this command. Project-local instructions (`.claude/conjure-config.md`) take precedence over global ones when they conflict. If neither file exists, proceed with defaults.

---

## Phase 1: Parse Arguments + Detect Mode

Parse `$ARGUMENTS`:

| Pattern | Mode |
|---------|------|
| empty | **Create** — ask what the hook should do |
| description only (no slug) | **Create** — use description as intent |
| `<slug>` only, found in settings.json | **Update** — read entry + script, refine |
| `<slug>` only, not found | **Create** — use slug as intended name |
| `<slug> review` | **Review** — audit script + settings entry, report issues |
| `<slug> add-event` | **Extend** — add a new event to an existing hook's script |
| `help` | Stop and print the Phase 1 argument table above, then stop |

**Finding an existing hook by slug:**
```bash
# Search settings.json for a hook whose script path or description contains the slug
python3 -c "
import json, sys
slug = sys.argv[1]
s = json.load(open(open(sys.argv[2]).name if len(sys.argv)>2 else open(sys.argv[1]).name))
" "<slug>" ~/.claude/settings.json 2>/dev/null

# Or simply grep for the script name
grep -r "<slug>" ~/.claude/settings.json ~/.claude/hooks/ 2>/dev/null | head -5
ls ~/.claude/hooks/*<slug>* 2>/dev/null
```

**If Update mode:** Read the settings.json entry AND the script file fully before proceeding to Phase 3.

**If Review mode:** Read both, then report:
- Is the event type correct for the stated purpose?
- Is the script executable (`chmod +x`)?
- Does it handle the blocking/non-blocking contract correctly?
- Are there output channel issues (Bash-matched hook bug)?
- Apply fixes, then output the Phase 4 report.

---

## Phase 2: Understand

The user's intent is: **$ARGUMENTS** (or clarified via intake)

If creating and intent is still unclear after parsing arguments, ask what the hook should do.

Read `~/.claude/settings.json` to understand existing hooks and their structure.

Determine:
- User-level hook (`~/.claude/`) or project-level (`.claude/`)?
- What event type, matcher, and handler type fit the intent?

## Phase 3: Design

Choose the right combination:

**Event → When it fires → Can block?**

| Event | Fires when | Blocks? | Common use |
|-------|-----------|---------|-----------|
| `SessionStart` | Session begins/resumes | No | Load context, set env vars |
| `SessionEnd` | Session terminates | No | Cleanup, logging |
| `Setup` | `--init` / `--maintenance` | No | One-time initialization |
| `UserPromptSubmit` | User hits enter | YES (exit 2) | Inject branch context, validate prompt |
| `UserPromptExpansion` | Slash command expands | YES (exit 2) | Intercept skill invocations |
| `PreToolUse` | Before any tool call | YES (exit 2) | Security guards, pattern injection |
| `PostToolUse` | After tool succeeds | No | Lint, format, log |
| `PostToolBatch` | Parallel tools all resolve | YES (exit 2 stops loop) | Validate batch results |
| `PostToolUseFailure` | After tool fails | No | Error logging |
| `PermissionRequest` | Permission dialog appears | YES | Auto-approve safe patterns |
| `PermissionDenied` | Permission auto-denied | No | Audit logging |
| `Stop` | Claude finishes responding | Can continue loop | Quality gates, notifications |
| `StopFailure` | Turn ends in API error | No | Error handling |
| `SubagentStart` | Subagent spawned | No | Audit, context injection |
| `SubagentStop` | Subagent finishes | Can continue | Validate subagent output |
| `TaskCreated` | Task created | No | Logging |
| `TaskCompleted` | Task completes | No | Post-task processing |
| `TeammateIdle` | Teammate goes idle | No | Coordination |
| `PreCompact` | Before compaction | YES (exit 2) | Snapshot state |
| `PostCompact` | After compaction | No | Restore context |
| `WorktreeCreate` | Worktree created | YES (any non-zero) | Bootstrap files |
| `WorktreeRemove` | Worktree removed | No | Cleanup |
| `Notification` | Claude sends notification | No | Forward to dashboard |
| `FileChanged` | Watched file changes | No | React to file edits |
| `CwdChanged` | Working directory changes | No | Update context |
| `ConfigChange` | settings.json changes | YES (exit 2) | Validate config |
| `InstructionsLoaded` | CLAUDE.md / rules file loads | No | Audit loaded rules |
| `Elicitation` | MCP server requests user input | YES | Intercept MCP prompts |
| `ElicitationResult` | User responds to elicitation | No | Log MCP interactions |

**Handler type restrictions:**
- `WorktreeCreate`, `SubagentStart`, `PreCompact` → only `command` or `http` (not `prompt` or `agent`)
- All other events → any handler type

**Matcher syntax:**
```
""                    → fires for ALL occurrences
"Bash"                → exact tool name match
"Write|Edit"          → pipe-separated list (NO spaces around |)
"startup"             → SessionStart source: startup|resume|clear|compact
"mcp__.*__write.*"    → JavaScript regex (any non-word char triggers regex mode)
```

**Handler types:** `command` (shell/Python script) · `http` (POST JSON to URL) · `prompt` (LLM yes/no) · `agent` (subagent with tools) · `mcp_tool` (invoke an MCP tool directly)

Use **Python** for complex logic, JSON manipulation, multi-file checks. Use **Bash** for simple greps or git commands.

## Phase 4: Execute

### 1. Write the script

For a Bash script (`~/.claude/hooks/<name>.sh`):

```bash
#!/bin/bash
INPUT=$(cat)
TOOL_NAME=$(echo "$INPUT" | jq -r '.tool_name // empty')
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')

# Available env vars: CLAUDE_PROJECT_DIR, CLAUDE_PLUGIN_ROOT, CLAUDE_EFFORT, CLAUDE_CODE_REMOTE
# SessionStart/Setup/CwdChanged/FileChanged also have CLAUDE_ENV_FILE

# Block (exit 2 + stderr):
# echo "Reason" >&2; exit 2

# Informational output — one-liner for UI, full detail for Claude:
# ⚠️  For Bash-matched hooks (matcher: "Bash"), additionalContext AND systemMessage AND stdout
#     are all dropped due to a known bug (#55889). Use permissionDecisionReason for PreToolUse
#     blocking decisions instead.
#
# For non-Bash-matched hooks:
# SUMMARY="[hook-name] found something"
# DETAIL="full\nmulti-line\ndetail"
# python3 -c "import json,sys; print(json.dumps({'systemMessage':sys.argv[1],'hookSpecificOutput':{'hookEventName':'PostToolUse','additionalContext':sys.argv[2]}}))" "$SUMMARY" "$DETAIL"

exit 0
```

For a Python script (`~/.claude/hooks/<name>.py`):

```python
#!/usr/bin/env python3
import json, sys

def main():
    try:
        data = json.loads(sys.stdin.read())
    except (json.JSONDecodeError, ValueError):
        return
    tool_input = data.get("tool_input", {})
    file_path = tool_input.get("file_path", "")

    # Block: print("reason", file=sys.stderr); sys.exit(2)

    # Informational: one-liner for UI, full detail injected for Claude
    # summary = "[hook-name] N issues in file.py"
    # detail = "full\nmulti-line\ncontent"
    # output = {"systemMessage": summary, "hookSpecificOutput": {"hookEventName": "PostToolUse", "additionalContext": detail}}
    # print(json.dumps(output))

main()
```

### 2. Make it executable

```bash
chmod +x ~/.claude/hooks/<name>.sh
```

### 2.5 Toggleable flag profile (optional)

Use this when a hook should be conditionally active — on in some sessions, off in others — without editing settings.json.

Add at the top of the script, after the INPUT read:
```bash
FLAGS_FILE="${HOME}/.claude/.hook-flags"
if ! grep -qs "enable-<hook-name>" "$FLAGS_FILE" 2>/dev/null; then
  exit 0  # not enabled — silently skip
fi
```

Manage flags:
```bash
echo "enable-<hook-name>" >> ~/.claude/.hook-flags   # turn on
sed -i '' "/enable-<hook-name>/d" ~/.claude/.hook-flags  # turn off
cat ~/.claude/.hook-flags  # list active flags
```

Use this pattern for: noisy diagnostic hooks, experiment hooks, hooks that don't apply to all projects.
Don't use it for: security guards, always-on formatters — those should always run.

### 3. Add the settings.json entry

Match the existing style — every entry needs `description` and `_source`:

```json
{
  "matcher": "Write|Edit",
  "description": "One-line description of what this hook does",
  "_source": "your-project",
  "if": "tool_input.file_path ends with .py",
  "hooks": [
    {
      "type": "command",
      "command": "bash",
      "args": ["${HOME}/.claude/hooks/<name>.sh"],
      "timeout": 10,
      "statusMessage": "Checking..."
    }
  ]
}
```

**Field notes:**
- `if` — optional condition string that narrows execution within a matcher (e.g. `"tool_input.file_path ends with .swift"`)
- `once` — if `true`, hook runs once per session. ⚠️ Only valid in **frontmatter-defined hooks** (inside a skill/agent `.md` file), NOT in `settings.json` entries.

**Timeout defaults:**
- `command` / `http` / `mcp_tool`: **600s** default
- `prompt`: **30s** default
- `agent`: **60s** default
- `UserPromptSubmit` (all handler types): **hardcapped at 30s** regardless of your `timeout` setting
- Guidance: `timeout: 3` for async logging · `timeout: 5–10` for PreToolUse/PostToolUse guards

**Async guidance:** `async: true` for logging/side-effects · `asyncRewake: true` for background work that may need to wake Claude — NEVER set both (`asyncRewake` implies `async`)

## Output Pattern Standard

**Every informational hook MUST follow the one-liner pattern:**
- `systemMessage` → one line shown in Claude Code UI to the user
- `hookSpecificOutput.additionalContext` → full content injected into Claude's model context (invisible in UI)

This keeps the UI clean while Claude still receives full context.

**⚠️ Bash-matched hooks bug (#55889):** For hooks matching the `"Bash"` tool, `additionalContext`, `systemMessage`, and plain stdout are ALL dropped (confirmed bug, open as of v2.1.141). Use `permissionDecisionReason` for PreToolUse blocking, or `exit 2` + stderr for hard blocks. There is no workaround for informational output on Bash-matched hooks.

### Python template (informational output, non-Bash-matched):
```python
import json, sys

summary = "[hook-name] 3 issues found in my_file.py"
detail = "Full multi-line\ncontent here"

output = {"systemMessage": summary}
if detail:
    output["hookSpecificOutput"] = {
        "hookEventName": "PostToolUse",  # match the event
        "additionalContext": detail,
    }
print(json.dumps(output))
```

### Bash template (informational output, non-Bash-matched):
```bash
SUMMARY="[hook-name] 2 violations in foo.py"
DETAIL="line 42: warning: ...\nline 55: error: ..."
python3 -c "
import json, sys
print(json.dumps({
    'systemMessage': sys.argv[1],
    'hookSpecificOutput': {'hookEventName': 'PostToolUse', 'additionalContext': sys.argv[2]}
}))" "$SUMMARY" "$DETAIL"
```

### When to stay silent:
- Side-effect-only hooks (logging, file writes) — exit 0, no output
- Bash-matched hooks when nothing notable happened — exit 0, no output

## JSON Output Reference

```json
// One-liner UI + full context for Claude (standard pattern):
{
  "systemMessage": "[hook-name] 3 issues in Foo.py",
  "hookSpecificOutput": {
    "hookEventName": "PostToolUse",
    "additionalContext": "Full detail content here\nLine 2\nLine 3"
  }
}

// Block Stop (keep conversation going):
{"decision": "block", "reason": "Fix violations before stopping"}

// PermissionRequest — allow or deny (must nest inside hookSpecificOutput):
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {"behavior": "allow"}
  }
}
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {"behavior": "deny"}
  }
}

// PreToolUse — deny with reason (exit 0, different schema from PermissionRequest):
{"hookSpecificOutput": {"hookEventName": "PreToolUse", "permissionDecision": "deny", "permissionDecisionReason": "reason"}}

// PreToolUse — allow + inject context (exit 0):
{"hookSpecificOutput": {"hookEventName": "PreToolUse", "permissionDecision": "allow", "additionalContext": "context"}}

// Inject env vars (SessionStart/Setup/CwdChanged/FileChanged only):
// echo "export MY_VAR=value" >> "$CLAUDE_ENV_FILE"
```

**Key distinction:** `PermissionRequest` uses `decision.behavior` nested in `hookSpecificOutput`. `PreToolUse` uses `permissionDecision` (flat string) in `hookSpecificOutput`. These are two different schemas — do not conflate.

Note: `additionalContext` is ONLY valid inside `hookSpecificOutput` — NOT at top level.

## Critical Gotchas

- **NEVER** set both `async: true` AND `asyncRewake: true` — asyncRewake implies async
- **NEVER** use `additionalContext` at top level — use `systemMessage` instead
- **Bash-matched hooks** — all output channels broken (additionalContext, systemMessage, stdout) — confirmed bug; use permissionDecisionReason or stderr+exit2 only
- **WorktreeCreate** — ANY non-zero exit aborts worktree creation
- **`UserPromptSubmit` timeout** — hardcapped at 30s regardless of your setting
- **Scripts MUST be `chmod +x`** — hook silently fails if not executable
- **Exit code 1 does NOT block** — only exit code 2 blocks (or any non-zero for WorktreeCreate)
- **Exec form** (args array) is safer than shell form — handles spaces in paths
- **`once: true`** — only valid in frontmatter hooks, NOT in settings.json entries

## Phase 5: Report

Output exactly 3 lines:
```
event: <EventType> / matcher: "<matcher>"
script: <path to script>
does: <one-line description of what it does>
```
