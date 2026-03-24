# cmux Integration

This session runs inside cmux, a native macOS terminal for AI coding agents built on the Ghostty engine. Check `$CMUX_WORKSPACE_ID` to confirm you're inside cmux before using cmux commands.

<env_vars>
| Variable | Purpose |
|----------|---------|
| `CMUX_WORKSPACE_ID` | Current workspace reference |
| `CMUX_SURFACE_ID` | Current surface (pane/tab) reference |
| `CMUX_SOCKET_PATH` | Unix socket for API calls |
</env_vars>

<sidebar>
```bash
cmux set-status <key> <value> [--icon <name>] [--color <#hex>]
cmux clear-status <key>
cmux set-progress <0.0-1.0> [--label <text>]
cmux clear-progress
cmux log [--level <level>] [--source <name>] -- <message>
cmux notify --title <text> [--subtitle <text>] [--body <text>]
```
</sidebar>

<pane_management>
```bash
cmux new-split <left|right|up|down>
cmux new-pane [--type <terminal|browser>] [--direction <left|right|up|down>] [--url <url>]
cmux send [--surface <ref>] <text>
cmux send-key [--surface <ref>] <key>
cmux read-screen [--surface <ref>] [--scrollback] [--lines <n>]
cmux list-panes | list-workspaces | tree [--all] | identify [--workspace <ref>]
```
</pane_management>

<browser>
All browser commands follow: `cmux browser [--surface <ref>] <subcommand> [args]`

The `--surface` flag goes BEFORE the subcommand — placing it after will error. Capture the `surface=surface:N` from `open-split` to target subsequent commands. When omitted, `$CMUX_SURFACE_ID` is used.

Subcommands (omitting the `cmux browser` prefix):

```
open [url]                          open-split [url]
goto|navigate <url>                 back | forward | reload
url|get-url

snapshot [-i] [--cursor] [--compact] [--max-depth <n>] [--selector <css>]
screenshot [--out <path>] [--json]
get <url|title|text|html|value|attr|count|box|styles>
is <visible|enabled|checked> <selector>
console <list|clear>                errors <list|clear>
highlight <selector>

click|dblclick|hover|focus|check|uncheck|scroll-into-view <selector>
type <selector> <text>              fill <selector> [text]
press|keydown|keyup <key>           select <selector> <value>
scroll [--selector <css>] [--dx <n>] [--dy <n>]
eval <script>
wait [--selector <css>] [--text <text>] [--url-contains <text>] [--load-state <state>] [--function <js>] [--timeout-ms <ms>]
find <role|text|label|placeholder|alt|title|testid|first|last|nth> ...

frame <selector|main>               dialog <accept|dismiss> [text]
download [wait] [--path <path>]      upload_file <selector> <path>

identify [--surface <ref>]           tab <new|list|switch|close|<index>>
cookies <get|set|clear>              storage <local|session> <get|set|clear>
state <save|load> <path>
addinitscript <script>               addscript <script>
addstyle <css>
```

Most interaction/navigation subcommands accept `--snapshot-after` to return a snapshot after the action.
</browser>

<teams_and_agents>
When spawning teams or agents, make them visible in cmux panes instead of running as invisible background subprocesses.

**CRITICAL: `cmux send` does NOT press Enter.** You must always follow `cmux send` with a separate `cmux send-key "Return"` to execute the command. This is the #1 source of stuck agents.

```bash
# CORRECT - two-step: type, then press Enter
cmux send --surface surface:N "some command"
cmux send-key --surface surface:N "Return"

# WRONG - command is typed but never executed
cmux send --surface surface:N "some command"
```

**Workflow — Interactive Claude Sessions (preferred):**
1. Name the lead/coordinator session first: `/rename commander` (or similar) so it's identifiable in `cmux tree`.
2. Create split panes — use `cmux new-split right` and `cmux new-split down` for a multi-pane layout (lead on left, agents on right stacked vertically).
3. Capture the `surface:N` reference returned by `cmux new-split`.
4. Launch claude with a descriptive `--name` in each pane:
   ```bash
   cmux send --surface surface:N "claude --dangerously-skip-permissions --name agent-name-here"
   cmux send-key --surface surface:N "Return"
   ```
   Use names matching the agent's role (e.g., `ui-ux-engineer`, `logic-dev`, `architect`).
5. Wait for claude to initialize (~5-10s), then send the task as a message:
   ```bash
   # For short prompts, send directly:
   cmux send --surface surface:N "your task description here"
   cmux send-key --surface surface:N "Return"

   # For long/complex prompts, write to a file first and tell the agent to read it:
   # (Write prompt to /tmp/agent-task.txt with the Write tool, then:)
   cmux send --surface surface:N "Read /tmp/agent-task.txt and follow the instructions in it"
   cmux send-key --surface surface:N "Return"
   ```
6. Monitor agent progress with `cmux read-screen --surface surface:N --lines <n>`.
   - Use `--scrollback` to see full history, not just the visible terminal area.
7. Check file changes to confirm agent output (e.g., `wc -l`, `Read`).
8. Use `cmux tree --all` to verify all agents are running and named correctly.

**Full example — launching a named agent:**
```bash
# 1. Create pane
cmux new-split right
# => OK surface:7 workspace:2

# 2. Launch claude with a name
cmux send --surface surface:7 "claude --dangerously-skip-permissions --name ui-ux-engineer"
cmux send-key --surface surface:7 "Return"
# (wait ~5-10s for claude to initialize)

# 3. Send the task
cmux send --surface surface:7 "Read /tmp/uiux-prompt.txt and follow the instructions in it"
cmux send-key --surface surface:7 "Return"
```

**Why interactive mode over `claude -p`:** Using `claude -p "..."` requires the entire prompt on the command line, which causes quoting/escaping nightmares when sent through `cmux send` (nested quotes, special characters, multi-line content all break). Interactive mode avoids this entirely — claude is already running, and `cmux send` just types the message into the session.

**Fallback — Agent tool:** If cmux pane agents aren't working (e.g., socket access issues, sandbox restrictions), fall back to the `Agent` tool for parallel work. Agents write to separate files so there are no conflicts. Use cmux `set-progress`/`set-status`/`notify` to keep the user informed of progress even when using invisible agents.

**Do NOT** spawn agents via the `Agent` tool with `team_name` when running inside cmux — they run as invisible background processes with no terminal visibility. Prefer cmux panes + `cmux send`/`cmux send-key` so the user can watch each agent work in real time.

**Layout patterns:**
- 2 agents: left (lead) + right (agent), or top (lead) + bottom (agent)
- 3 agents: left (lead) + top-right (agent 1) + bottom-right (agent 2)
- Browser preview: add a `--type browser --direction down` pane for live preview

**Progress tracking:**
- Use `cmux set-progress` to show overall team progress in the status bar.
- Use `cmux set-status` to show which agents are active and their current phase.
- Use `cmux notify` when an agent completes its work or hits a blocker.

**Cleanup:**
- Agents launched via `claude -p` exit automatically when done.
- Interactive agents need `/exit` or Ctrl+C sent via `cmux send-key`.
- Close extra panes with `cmux close-surface --surface surface:N` after work is complete.
</teams_and_agents>

<guidelines>
- Prefer `open-split` for web previews; prefer `right` or `down` splits.
- Prefer `snapshot` over `screenshot` for programmatic inspection (returns accessibility tree).
- Use `set-progress` for long multi-step tasks; `log` for milestones; `notify` for alerts needing attention across panes.
- Use `--snapshot-after` to chain inspection with interaction in a single call.
- When using teams/agents, always spawn them in visible cmux panes so the user can observe their work.
</guidelines>
